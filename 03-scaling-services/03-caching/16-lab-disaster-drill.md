# 3.3.16 — 🧪 Lab: The Disaster Drill

> **Part 3 · Scaling Services · Caching · Chapter 16 of 18**
> Cause each failure mode on purpose. Watch the origin die. Apply the fix. Watch it survive.

---

## 🧒 ELI5 — Explain Like I'm 5

You've read about all the ways the fridge system breaks. Now **break it on purpose, in your own kitchen, where it's safe.**

You will:

- **Unplug the fridge during dinner rush** and watch the shop get flooded.
- **Ask for purple milk a thousand times** and watch every request go to the shop.
- **Fill the fridge all at once** so everything expires together, then watch the cliff.
- **Turn away some customers at the door** and see the shop survive when it otherwise wouldn't.

The point isn't the code. The point is **seeing the numbers**: how many requests reach the shop, and whether the shop stays standing.

Once you've watched a database fall over because of a cache, you never design one carelessly again.

---

## Setup

```python
# disaster.py
import redis, time, threading, random, json
from collections import Counter

r = redis.Redis(decode_responses=True, socket_timeout=0.05)
stats = Counter()

class Origin:
    """A database with a hard concurrency limit — it fails past capacity."""
    def __init__(self, capacity=50, latency=0.02):
        self.sem = threading.Semaphore(capacity)
        self.latency, self.capacity = latency, capacity
        self.inflight = 0
        self._lock = threading.Lock()

    def query(self, pid):
        if not self.sem.acquire(blocking=False):
            stats["origin_rejected"] += 1
            raise RuntimeError("origin overloaded")     # what a real DB does
        try:
            with self._lock:
                self.inflight += 1
                stats["max_inflight"] = max(stats["max_inflight"], self.inflight)
            stats["origin_queries"] += 1
            time.sleep(self.latency)
            return {"id": pid, "name": f"product-{pid}"}
        finally:
            with self._lock:
                self.inflight -= 1
            self.sem.release()

db = Origin(capacity=50)

def zipf(n=10_000, a=1.2):
    return min(n, int(random.paretovariate(a - 1)))

def load(fn, n=3000, threads=100, id_fn=zipf):
    stats.clear()
    def worker(count):
        for _ in range(count):
            try:
                fn(id_fn())
                stats["success"] += 1
            except Exception:
                stats["failed"] += 1
    ts = [threading.Thread(target=worker, args=(n // threads,)) for _ in range(threads)]
    t0 = time.time()
    for t in ts: t.start()
    for t in ts: t.join()
    dur = time.time() - t0
    print(f"  {stats['success']:5d} ok, {stats['failed']:5d} failed | "
          f"origin: {stats['origin_queries']:5d} queries, "
          f"{stats['origin_rejected']:5d} rejected | {dur:.1f}s")
```

---

## Disaster 1 — Stampede

```python
def naive_get(pid):
    key = f"p:{pid}"
    if (v := r.get(key)) is not None:
        stats["hits"] += 1
        return json.loads(v)
    row = db.query(pid)
    r.set(key, json.dumps(row), ex=60)
    return row

# Warm the cache fully, then expire ONE very hot key
r.flushdb()
for pid in range(1, 200):
    naive_get(pid)
print("baseline (warm):")
load(naive_get, n=3000)

r.delete("p:1")          # the hottest key under Zipf
print("after hot key expiry:")
load(lambda _: naive_get(1), n=2000, threads=200)
```

**Typical result:**
```
baseline (warm):
   3000 ok,     0 failed | origin:    12 queries,     0 rejected | 0.4s
after hot key expiry:
   1180 ok,   820 failed | origin:   380 queries,   820 rejected | 2.9s
```

☠️ **Check:** one key expiring caused **820 failed requests**. The origin rejected them because 200 threads all tried to load the same value at once.

### Fix: coalescing

```python
def coalesced_get(pid):
    key = f"p:{pid}"
    if (v := r.get(key)) is not None:
        stats["hits"] += 1
        return json.loads(v)
    lock = f"lk:{key}"
    if r.set(lock, "1", nx=True, ex=5):
        try:
            row = db.query(pid)
            r.set(key, json.dumps(row), ex=60)
            return row
        finally:
            r.delete(lock)
    for _ in range(100):
        time.sleep(0.01)
        if (v := r.get(key)) is not None:
            stats["coalesced"] += 1
            return json.loads(v)
    return db.query(pid)              # bounded fallback

r.delete("p:1")
print("with coalescing:")
load(lambda _: coalesced_get(1), n=2000, threads=200)
```

**Typical result:**
```
   2000 ok,     0 failed | origin:     1 queries,     0 rejected | 0.6s
```

✅ **Check: 380 origin queries → 1. Zero failures.** One line of locking turned a partial outage into a non-event.

---

## Disaster 2 — Penetration

```python
r.flushdb()
print("requests for non-existent IDs:")
load(naive_get, n=2000, threads=100, id_fn=lambda: random.randint(10**7, 10**8))
```

**Typical result:**
```
    980 ok,  1020 failed | origin:   980 queries,  1020 rejected | 1.9s
```

☠️ **Check: a 0% hit rate.** Every request reached the origin. The cache provided **no protection at all**. An attacker can trivially bypass your cache by requesting random IDs.

### Fix: negative caching

```python
NULL = "\x00NULL"

def negative_cached_get(pid):
    key = f"p:{pid}"
    v = r.get(key)
    if v == NULL:
        stats["neg_hits"] += 1
        return None
    if v is not None:
        stats["hits"] += 1
        return json.loads(v)
    row = db.query(pid) if pid <= 10_000 else None
    r.set(key, json.dumps(row) if row else NULL, ex=60 if row else 15)
    return row

r.flushdb()
print("with negative caching (100 distinct bad IDs):")
load(negative_cached_get, n=2000, threads=100,
     id_fn=lambda: random.randint(10**7, 10**7 + 100))
```

**Typical result:**
```
   2000 ok,     0 failed | origin:   100 queries,     0 rejected | 0.5s
```

✅ **Check: 980 → 100 origin queries**, and zero failures. The 100 distinct bad IDs were each looked up once and then remembered as absent.

⚠️ Now re-run with **unbounded** random IDs (`randint(10**7, 10**9)`). ✅ **Check:** negative caching helps much less, because every ID is distinct. **This is where a Bloom filter or input validation is required** — negative caching only helps when the bad IDs repeat.

---

## Disaster 3 — Avalanche (synchronised expiry)

```python
r.flushdb()
# Bulk warm 2,000 keys with IDENTICAL TTLs
for pid in range(1, 2001):
    r.set(f"p:{pid}", json.dumps({"id": pid}), ex=4)

print("t=0 (warm):"); load(naive_get, n=2000)
time.sleep(4.2)                    # everything expires at once
print("t=4.2 (all expired):"); load(naive_get, n=2000)
```

**Typical result:**
```
t=0 (warm):
   2000 ok,     0 failed | origin:     0 queries,     0 rejected | 0.2s
t=4.2 (all expired):
   1240 ok,   760 failed | origin:   620 queries,   760 rejected | 3.1s
```

☠️ **Check:** the cache went from perfect to useless in one instant, and the origin couldn't cope.

### Fix: TTL jitter

```python
r.flushdb()
for pid in range(1, 2001):
    r.set(f"p:{pid}", json.dumps({"id": pid}),
          ex=int(4 * random.uniform(0.7, 1.3)))     # ← 2.8s to 5.2s spread

time.sleep(4.2)
print("with jitter:"); load(naive_get, n=2000)
```

**Typical result:**
```
   2000 ok,     0 failed | origin:   210 queries,     0 rejected | 0.7s
```

✅ **Check:** expiry spread over a 2.4-second window instead of one instant. The origin absorbed the misses comfortably. **One `random.uniform()` call.**

---

## Disaster 4 — Total cache loss

```python
r.flushdb()
for pid in range(1, 500): naive_get(pid)       # warm
print("warm baseline:"); load(naive_get, n=3000)

r.flushdb()                                     # 💀 the cache is gone
print("cache flushed, full traffic:"); load(naive_get, n=3000)
```

**Typical result:**
```
warm baseline:
   3000 ok,     0 failed | origin:    20 queries,     0 rejected | 0.5s
cache flushed, full traffic:
    890 ok,  2110 failed | origin:   760 queries,  2110 rejected | 4.8s
```

☠️ **Check: 70% of requests failed.** This is the number that matters. A cache flush — one command — took down a system that was completely healthy a second earlier.

### Fix A: local cache (the shock absorber)

```python
from threading import Lock
_local, _local_lock = {}, Lock()
LOCAL_TTL = 1.0

def local_then_redis(pid):
    now = time.time()
    with _local_lock:
        e = _local.get(pid)
        if e and e[1] > now:
            stats["local_hits"] += 1
            return e[0]
    row = coalesced_get(pid)                       # falls through to Redis, then origin
    with _local_lock:
        _local[pid] = (row, now + LOCAL_TTL)
    return row

r.flushdb(); _local.clear()
print("cold cache + local cache + coalescing:"); load(local_then_redis, n=3000)
```

**Typical result:**
```
   3000 ok,     0 failed | origin:   145 queries,     0 rejected | 1.1s
```

✅ **Check: 2,110 failures → 0.** A 1-second local cache plus coalescing absorbed a total cache loss at full traffic. **This is the single most valuable defence in the chapter.**

### Fix B: load shedding

```python
class Shedder:
    def __init__(self, limit): self.limit, self.n, self.lock = limit, 0, Lock()
    def __enter__(self):
        with self.lock:
            if self.n >= self.limit:
                stats["shed"] += 1
                raise RuntimeError("shed")
            self.n += 1
    def __exit__(self, *a):
        with self.lock: self.n -= 1

shedder = Shedder(limit=40)     # below the origin's capacity of 50

def shed_get(pid):
    key = f"p:{pid}"
    if (v := r.get(key)) is not None:
        stats["hits"] += 1
        return json.loads(v)
    with shedder:                                  # only 40 concurrent origin loads
        row = db.query(pid)
        r.set(key, json.dumps(row), ex=60)
        return row

r.flushdb()
print("cold cache + load shedding:"); load(shed_get, n=3000)
```

**Typical result:**
```
   1420 ok,  1580 failed | origin:  1420 queries,     0 rejected | 2.2s
                                                     ↑ zero origin rejections
```

✅ **Check:** more requests failed than with fix A — **but the origin was never overloaded**. Shedding trades some requests for guaranteed origin survival. In production the shed requests would be the *low-priority* ones (bots, anonymous users), and the origin would keep serving checkout traffic.

⚖️ **Compare the two fixes:** the local cache *avoids* the load; shedding *survives* it. Use both — the local cache first, shedding as the backstop.

---

## Disaster 5 — Big key

```python
r.flushdb()
big = "x" * (50 * 1024 * 1024)            # 50 MB
r.set("big", big)

def timed_small_get():
    t0 = time.time(); r.get("p:1"); return (time.time() - t0) * 1000

r.set("p:1", "small")
print(f"small GET, idle:  {timed_small_get():.2f} ms")

def hammer_big():
    for _ in range(5): r.get("big")

t = threading.Thread(target=hammer_big); t.start()
time.sleep(0.02)
print(f"small GET, during big reads: {timed_small_get():.2f} ms")
t.join()
```

**Typical result:**
```
small GET, idle:  0.31 ms
small GET, during big reads: 84.20 ms
```

☠️ **Check: a 270× latency increase on an unrelated key.** Redis is single-threaded — one 50 MB value blocks **every** client. Your p99 across the whole service just collapsed because of one oversized cache entry.

```bash
redis-cli --bigkeys      # find these before production does
```

✅ **Fix:** split it, compress it, or use the claim-check pattern (store it in blob storage, cache the key). And use `UNLINK` rather than `DEL` to delete large values asynchronously.

---

## Disaster 6 — The silent hit-rate collapse

```python
r.flushdb()
for pid in range(1, 500): naive_get(pid)          # written with prefix "p:"

def deployed_with_typo(pid):                       # a "new version" reads "prod:"
    key = f"prod:{pid}"                            # ← different prefix
    if (v := r.get(key)) is not None:
        stats["hits"] += 1
        return json.loads(v)
    row = db.query(pid)
    r.set(key, json.dumps(row), ex=60)
    return row

print("after key-format change:"); load(deployed_with_typo, n=2000)
```

**Typical result:**
```
   1180 ok,   820 failed | origin:   480 queries,   820 rejected | 2.6s
```

☠️ **Check:** no error was raised anywhere. No exception, no alert on cache health — Redis is perfectly healthy and reports 100% availability. The system is simply **10× more expensive and 10× slower**, silently.

✅ **Detection:**
```python
hits, misses = stats["hits"], stats["origin_queries"]
hit_rate = hits / max(hits + misses, 1)
if hit_rate < BASELINE_HIT_RATE - 0.10:
    alert("cache hit rate dropped >10pp — check key format / deploy")
```

**Alert on hit rate as a first-class SLI.** It is the only thing that catches this class of bug.

---

## Summary of what you measured

| Disaster | Without the fix | With the fix |
|---|---|---|
| Stampede | 380 origin queries, 820 failures | 1 query, 0 failures |
| Penetration | 980 queries, 0% hit rate | 100 queries, 0 failures |
| Avalanche | 620 queries, 760 failures | 210 queries, 0 failures |
| Total cache loss | 2,110 failures (70%) | 0 failures (local cache) |
| Big key | 0.31 ms → 84 ms on unrelated keys | Split / claim check |
| Silent collapse | Invisible 10× cost increase | Hit-rate alert |

---

## 📋 Lab checklist

| Drill | Done? |
|---|---|
| 1 Caused and fixed a stampede | ☐ |
| 2 Caused and fixed penetration; found negative caching's limit | ☐ |
| 3 Caused and fixed an avalanche with jitter | ☐ |
| 4 Survived total cache loss with a local cache | ☐ |
| 4b Survived it a second way, with load shedding | ☐ |
| 5 Measured big-key blocking | ☐ |
| 6 Reproduced a silent hit-rate collapse | ☐ |

**The sentence to take away:** *every one of those fixes is under ten lines of code, and every one is the difference between a slow minute and an outage.*

---

**← Previous** [3.3.15 Failure Modes](15-failure-modes.md)
**Next →** [3.3.17 Security & Observability](17-security-and-observability.md)
