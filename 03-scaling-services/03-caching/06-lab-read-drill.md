# 3.3.6 — 🧪 Lab: The Read Drill

> **Part 3 · Scaling Services · Caching · Chapter 6 of 18**
> Build a cache, measure the hit rate, cause a stampede, then fix it. Numbers you produce yourself stick.

---

## 🧒 ELI5 — Explain Like I'm 5

Time to actually build the fridge.

You'll:

1. Put milk in a fridge and **count how many trips to the shop you save.**
2. Watch what happens when **the whole street's milk runs out at the same second** — everyone runs to the shop at once and the shop falls over. You'll see the number of shop trips explode.
3. Fix it by making **one person go, while everyone else waits.**
4. Fix it a second way, by having everyone **check their milk at slightly different times** so they never all run out together.
5. Discover that **asking for milk that doesn't exist** goes to the shop every single time — and fix that too.

Every fix here is three lines of code. The point is to *see the number change*.

---

## Setup

```bash
docker run -d --name cache-lab -p 6379:6379 redis:7-alpine
pip install redis
```

```python
# lab.py — a fake "slow database" and instrumentation
import redis, time, random, threading, json, math
from collections import Counter

r = redis.Redis(decode_responses=True)
stats = Counter()

DB = {i: {"id": i, "name": f"product-{i}", "price": i * 100} for i in range(1, 10_001)}

def slow_db_get(pid):
    """Simulates a 50 ms indexed query."""
    stats["db_queries"] += 1
    time.sleep(0.05)
    return DB.get(pid)

def zipf_id(n=10_000, a=1.2):
    """Realistic skewed access: a few items get most of the traffic."""
    return min(n, int(random.paretovariate(a - 1)))
```

---

## Drill 1 — Measure the hit rate with realistic skew

```python
def get_cache_aside(pid, ttl=60):
    key = f"prod:v1:{pid}"
    if (v := r.get(key)) is not None:
        stats["hits"] += 1
        return json.loads(v)
    stats["misses"] += 1
    row = slow_db_get(pid)
    if row:
        r.set(key, json.dumps(row), ex=ttl)
    return row

r.flushdb(); stats.clear()
t0 = time.time()
for _ in range(2000):
    get_cache_aside(zipf_id())
elapsed = time.time() - t0

total = stats["hits"] + stats["misses"]
print(f"hit rate    : {stats['hits']/total:.1%}")
print(f"db queries  : {stats['db_queries']}")
print(f"total time  : {elapsed:.1f}s")
print(f"avg latency : {elapsed/total*1000:.1f}ms")
print(f"distinct keys: {r.dbsize()}")
```

**Typical result:**
```
hit rate    : 88.3%
db queries  : 234
total time  : 12.1s
avg latency : 6.1ms
distinct keys: 234
```

✅ **Check:** 2,000 requests produced only ~234 database queries. **That's the whole value of caching in one number.**

### 🧪 Now prove that skew is what makes it work

```python
r.flushdb(); stats.clear()
for _ in range(2000):
    get_cache_aside(random.randint(1, 10_000))    # UNIFORM, not Zipf
total = stats["hits"] + stats["misses"]
print(f"uniform hit rate: {stats['hits']/total:.1%}")
```

**Typical result:** `uniform hit rate: 9.8%`

✅ **Check:** the same cache, the same size, the same code — and the hit rate collapsed from 88% to 10%. **Caching works because of access skew, not because caches are magic.** If someone tells you a cache will fix a uniformly-random access pattern, you now have the number to disagree with.

---

## Drill 2 — Cause a stampede

```python
def concurrent_load(fn, key_id, n_threads=200):
    stats.clear()
    barrier = threading.Barrier(n_threads)
    def worker():
        barrier.wait()                # all start at the same instant
        fn(key_id)
    threads = [threading.Thread(target=worker) for _ in range(n_threads)]
    t0 = time.time()
    for t in threads: t.start()
    for t in threads: t.join()
    return time.time() - t0, stats["db_queries"]

r.flushdb()
get_cache_aside(42)                   # warm it
r.delete("prod:v1:42")                # simulate expiry of a HOT key

elapsed, queries = concurrent_load(get_cache_aside, 42)
print(f"naive cache-aside: {queries} db queries in {elapsed:.2f}s")
```

**Typical result:**
```
naive cache-aside: 187 db queries in 0.31s
```

☠️ **Check:** one key expired and **187 identical queries** hit the database in a third of a second. In production this is a key receiving 10,000 QPS: the moment its TTL lapses, your database receives a burst of thousands of identical queries. **This is how caches cause outages.**

---

## Drill 3 — Fix it with a distributed lock (coalescing)

```python
def get_coalesced(pid, ttl=60):
    key = f"prod:v1:{pid}"
    if (v := r.get(key)) is not None:
        stats["hits"] += 1
        return json.loads(v)

    lock_key = f"lock:{key}"
    if r.set(lock_key, "1", nx=True, ex=10):          # leader
        try:
            row = slow_db_get(pid)
            if row:
                r.set(key, json.dumps(row), ex=ttl)
            return row
        finally:
            r.delete(lock_key)
    else:                                              # follower
        for _ in range(100):
            time.sleep(0.01)
            if (v := r.get(key)) is not None:
                stats["coalesced"] += 1
                return json.loads(v)
        stats["lock_timeout"] += 1
        return slow_db_get(pid)                        # fallback: never deadlock

r.flushdb()
get_coalesced(42); r.delete("prod:v1:42")
elapsed, queries = concurrent_load(get_coalesced, 42)
print(f"coalesced: {queries} db queries in {elapsed:.2f}s, "
      f"{stats['coalesced']} waited, {stats['lock_timeout']} timed out")
```

**Typical result:**
```
coalesced: 1 db queries in 0.09s, 199 waited, 0 timed out
```

✅ **Check: 187 → 1.** One database query served 200 concurrent requests, and it was *faster* overall because the database wasn't overloaded.

### 🧪 Sub-drill: what if the leader dies?

Comment out the `finally: r.delete(lock_key)` and kill the leader mid-load. ✅ **Check:** followers spin until the lock's own 10 s TTL expires, then one becomes leader. **This is why the lock needs a TTL** — without `ex=10`, every subsequent request would block forever.

---

## Drill 4 — Fix it with probabilistic early expiration

No locks at all.

```python
def get_xfetch(pid, ttl=60, beta=1.0):
    key = f"prod:v1:{pid}"
    raw = r.get(key)
    if raw is not None:
        entry = json.loads(raw)
        ttl_left = r.ttl(key)
        # refresh early with rising probability as expiry approaches
        if ttl_left > 0 and -entry["delta"] * beta * math.log(random.random()) >= ttl_left:
            raw = None                       # fall through and recompute
        else:
            stats["hits"] += 1
            return entry["v"]
    t0 = time.time()
    row = slow_db_get(pid)
    delta = time.time() - t0                 # how expensive this key is to compute
    if row:
        r.set(key, json.dumps({"v": row, "delta": delta}), ex=ttl)
    return row
```

Run the same 200-thread barrier test **and** a sustained load over several TTL cycles:

```python
r.flushdb(); stats.clear()
for _ in range(3000):
    get_xfetch(zipf_id())
print(f"xfetch db queries over 3000 reqs: {stats['db_queries']}")
```

✅ **Check:** database queries stay low and, crucially, **spread out over time** rather than spiking at TTL boundaries. Compare against plain cache-aside with the same TTL and watch the spikes disappear.

⚖️ **Coalescing vs xfetch:** coalescing is a hard guarantee (exactly one loader) but needs a lock and a fallback. xfetch is probabilistic and lock-free, and it naturally refreshes expensive keys earlier. **Real systems often use both**: xfetch to avoid most simultaneous expiries, coalescing as the backstop.

---

## Drill 5 — TTL jitter

```python
r.flushdb(); stats.clear()
# Populate 1,000 keys at the same instant with the SAME TTL
for pid in range(1, 1001):
    r.set(f"prod:v1:{pid}", json.dumps(DB[pid]), ex=5)

time.sleep(5.1)                                  # they ALL expire together
t0 = time.time()
for pid in range(1, 1001):
    get_cache_aside(pid, ttl=5)
print(f"no jitter : {stats['db_queries']} queries in {time.time()-t0:.1f}s")

# Now with jitter
r.flushdb(); stats.clear()
for pid in range(1, 1001):
    r.set(f"prod:v1:{pid}", json.dumps(DB[pid]),
          ex=int(5 * random.uniform(0.8, 1.2)))   # ← ±20% jitter
```

✅ **Check:** without jitter, 1,000 keys expire in the same second and produce a synchronised burst. With jitter, expiry spreads over a 2-second window, and the database sees a gentle ramp instead of a cliff. **One line of code.**

This matters most after a **cache warm-up or a deploy**, when large numbers of keys are written at nearly the same moment — they then expire at nearly the same moment, forever, in lockstep.

---

## Drill 6 — Cache penetration and negative caching

```python
r.flushdb(); stats.clear()
for _ in range(500):
    get_cache_aside(random.randint(50_000, 60_000))   # IDs that do not exist
print(f"no negative caching: {stats['db_queries']} db queries / 500 requests")
```

**Typical result:** `no negative caching: 500 db queries / 500 requests` — a **0% hit rate**.

☠️ **Check:** every request for a non-existent key hits the database. An attacker requesting random IDs bypasses your cache completely. That is **cache penetration**.

```python
NULL = "\x00__NULL__"

def get_with_negative(pid, ttl=60, neg_ttl=15):
    key = f"prod:v1:{pid}"
    v = r.get(key)
    if v == NULL:
        stats["neg_hits"] += 1
        return None
    if v is not None:
        stats["hits"] += 1
        return json.loads(v)
    row = slow_db_get(pid)
    r.set(key, json.dumps(row) if row else NULL, ex=ttl if row else neg_ttl)
    return row

r.flushdb(); stats.clear()
for _ in range(500):
    get_with_negative(random.randint(50_000, 50_100))   # 100 distinct missing IDs
print(f"with negative caching: {stats['db_queries']} db queries, "
      f"{stats['neg_hits']} negative hits")
```

**Typical result:** `with negative caching: 100 db queries, 400 negative hits`

✅ **Check: 500 → 100.** And note the shorter TTL on negatives (15 s vs 60 s) so a newly-created record isn't invisible for a full minute.

---

## Drill 7 — Batch reads

```python
def get_one_by_one(ids):
    return [get_cache_aside(i) for i in ids]

def get_batched(ids, ttl=60):
    keys = [f"prod:v1:{i}" for i in ids]
    values = r.mget(keys)
    out, missing = {}, []
    for pid, raw in zip(ids, values):
        if raw is not None:
            out[pid] = json.loads(raw); stats["hits"] += 1
        else:
            missing.append(pid)
    if missing:
        stats["db_queries"] += 1                  # ONE query for all of them
        time.sleep(0.05)
        pipe = r.pipeline()
        for pid in missing:
            if (row := DB.get(pid)):
                out[pid] = row
                pipe.set(f"prod:v1:{pid}", json.dumps(row), ex=ttl)
        pipe.execute()
    return [out.get(i) for i in ids]

ids = list(range(1, 101))
r.flushdb(); stats.clear()
t0 = time.time(); get_one_by_one(ids); t1 = time.time()
r.flushdb(); stats.clear()
t2 = time.time(); get_batched(ids);    t3 = time.time()
print(f"one-by-one: {t1-t0:.2f}s   batched: {t3-t2:.2f}s")
```

**Typical result:** `one-by-one: 5.12s   batched: 0.06s` — roughly **85×**.

✅ **Check:** 100 round trips + 100 queries became 1 `MGET` + 1 query + 1 pipelined write. The N+1 problem exists in caches exactly as it does in databases, and it's just as expensive.

---

## Drill 8 — Hit rate vs cache size

```python
r.config_set("maxmemory", "2mb")
r.config_set("maxmemory-policy", "allkeys-lru")

for size in ["1mb", "2mb", "5mb", "20mb"]:
    r.config_set("maxmemory", size)
    r.flushdb(); stats.clear()
    for _ in range(5000):
        get_cache_aside(zipf_id())
    total = stats["hits"] + stats["misses"]
    print(f"{size:>5}: hit rate {stats['hits']/total:5.1%}  "
          f"keys {r.dbsize():5d}  evicted {r.info()['evicted_keys']}")
```

✅ **Check:** the hit-rate curve is **steep then flat**. Going from 1 MB to 5 MB buys a lot; going from 5 MB to 20 MB buys very little — because the Zipf tail is enormous and each additional item is requested less often.

🎯 **This curve is how you size a cache.** Find the knee, add ~30% headroom, and stop. Buying memory past the knee is spending money for nothing. See [Eviction & Sizing](11-eviction-and-sizing.md).

---

## 📋 Lab checklist

| Drill | Done? | Number you produced |
|---|---|---|
| 1 Hit rate with skew | ☐ | ~88% vs ~10% uniform |
| 2 Stampede | ☐ | ~187 queries from one expiry |
| 3 Coalescing | ☐ | 187 → 1 |
| 4 Probabilistic expiry | ☐ | Spread instead of spikes |
| 5 TTL jitter | ☐ | Cliff → ramp |
| 6 Negative caching | ☐ | 500 → 100 |
| 7 Batching | ☐ | ~85× faster |
| 8 Size vs hit rate | ☐ | Found the knee |

**The one sentence to take away:** *a cache is not just a speed-up — it changes the shape of your database load, including when it goes wrong.*

---

**← Previous** [3.3.5 Read Patterns](05-read-patterns.md)
**Next →** [3.3.7 Write Patterns: Mutating Data](07-write-patterns.md)
