# 3.3.9 — 🧪 Lab: The Write Drill

> **Part 3 · Scaling Services · Caching · Chapter 9 of 18**
> Reproduce every cache-consistency race on purpose, then fix each one and prove the fix works.

---

## 🧒 ELI5 — Explain Like I'm 5

Last lab you filled the fridge. This lab you **break** it — deliberately.

You'll make the fridge hold the wrong milk in four different ways:

1. Two people update it at once, and the **slower one wins**.
2. Someone fetches old milk, and puts it in **after** someone else cleared the fridge.
3. Someone puts milk in the fridge *before* the shop confirmed the change — and the shop change **fails**, so the fridge has milk that doesn't exist.
4. Someone fetches from the **slow copy of the shop's list** and caches an out-of-date answer.

For each one you'll see the wrong value with your own eyes, apply the fix, and see it become right.

**Reading about races doesn't teach you races. Reproducing them does.**

---

## Setup

```python
# writedrill.py
import redis, time, threading, json, random

r = redis.Redis(decode_responses=True)

class FakeDB:
    """A 'database' with a settable read latency, plus a lagging replica."""
    def __init__(self):
        self.data, self.version = {}, {}
        self.replica, self.lag = {}, 0.0
        self._lock = threading.Lock()

    def write(self, k, v):
        with self._lock:
            self.data[k] = v
            self.version[k] = self.version.get(k, 0) + 1
            ver = self.version[k]
        threading.Timer(self.lag, self._replicate, (k, v)).start()
        return ver

    def _replicate(self, k, v):
        self.replica[k] = v

    def read(self, k, delay=0.0, from_replica=False):
        time.sleep(delay)
        return (self.replica if from_replica else self.data).get(k), self.version.get(k, 0)

db = FakeDB()
KEY = lambda k: f"prod:v1:{k}"
```

---

## Drill 1 — The writer-vs-writer race (SET instead of DEL)

```python
def update_with_set(k, v, set_delay):
    db.write(k, v)
    time.sleep(set_delay)                 # simulate a slow path to the cache
    r.set(KEY(k), v, ex=300)

r.flushdb()
db.data.clear(); db.version.clear()

a = threading.Thread(target=update_with_set, args=("p1", "v1", 0.30))  # slow writer
b = threading.Thread(target=update_with_set, args=("p1", "v2", 0.05))  # fast writer

a.start(); time.sleep(0.01); b.start()
a.join(); b.join()

print(f"DB    = {db.data['p1']}")
print(f"CACHE = {r.get(KEY('p1'))}")
```

**Result:**
```
DB    = v1        ← whichever wrote last to the DB
CACHE = v1
```
Run it a few times, and adjust delays so the *fast* writer commits second:

```python
a = threading.Thread(target=update_with_set, args=("p1", "v1", 0.30))
b = threading.Thread(target=update_with_set, args=("p1", "v2", 0.01))
# b writes DB second (v2) but sets cache FIRST; a sets cache later with v1
```

☠️ **Check:** you will reach a state where `DB = v2` but `CACHE = v1`. The cache is wrong and stays wrong for the full 300 s TTL.

### The fix: DELETE instead of SET

```python
def update_with_del(k, v, delay):
    db.write(k, v)
    time.sleep(delay)
    r.delete(KEY(k))                      # delete is idempotent and order-independent

r.flushdb()
a = threading.Thread(target=update_with_del, args=("p1", "v1", 0.30))
b = threading.Thread(target=update_with_del, args=("p1", "v2", 0.01))
a.start(); b.start(); a.join(); b.join()
print(f"DB={db.data['p1']}  CACHE={r.get(KEY('p1'))}")
```

✅ **Check:** `CACHE = None`. No matter what order the deletes land in, the result is the same: empty. The next read repopulates from the database, correctly.

**Delete is commutative. Set is not.** That is the entire argument.

---

## Drill 2 — The stale-set-after-delete race (reader vs writer)

The race that delete alone cannot fix.

```python
def slow_reader(k, read_delay=0.30):
    if (v := r.get(KEY(k))) is not None:
        return v
    val, _ = db.read(k, delay=read_delay)     # slow read — the window
    r.set(KEY(k), val, ex=300)                # writes what it read, possibly stale
    return val

r.flushdb()
db.write("p2", "old")
r.delete(KEY("p2"))                            # start with a cold cache

reader = threading.Thread(target=slow_reader, args=("p2",))
reader.start()
time.sleep(0.05)                               # reader has read "old", hasn't SET yet

db.write("p2", "new")                          # writer commits
r.delete(KEY("p2"))                            # writer invalidates (cache is empty)

reader.join()                                  # reader now SETs "old"
print(f"DB={db.data['p2']}  CACHE={r.get(KEY('p2'))}")
```

**Result:**
```
DB=new  CACHE=old      ← wrong for 300 seconds
```

☠️ **Check:** everyone behaved correctly. Delete-on-write did not help, because the cache was already empty when the writer deleted. **This is the race that requires a real fix.**

### Fix A — Delayed double delete

```python
r.flushdb()
db.write("p3", "old"); r.delete(KEY("p3"))

reader = threading.Thread(target=slow_reader, args=("p3",))
reader.start(); time.sleep(0.05)

db.write("p3", "new")
r.delete(KEY("p3"))
threading.Timer(0.5, lambda: r.delete(KEY("p3"))).start()   # ← the second delete

reader.join(); time.sleep(0.6)
print(f"DB={db.data['p3']}  CACHE={r.get(KEY('p3'))}")
```

✅ **Check:** `CACHE = None`. The second delete landed after the reader's set.

⚠️ Now set the reader's delay to `1.0` (longer than the 0.5 s second-delete timer) and re-run. ✅ **Check:** the bug returns. **The delay is a guess.** Choose it from your p99 read latency, and understand it's a mitigation, not a guarantee.

### Fix B — Versioned writes (the correct fix)

```python
VERSIONED_SET = r.register_script("""
local cur = redis.call('HGET', KEYS[1], 'ver')
if cur == false or tonumber(ARGV[1]) > tonumber(cur) then
  redis.call('HSET', KEYS[1], 'ver', ARGV[1], 'val', ARGV[2])
  redis.call('EXPIRE', KEYS[1], ARGV[3])
  return 1
end
return 0
""")

def versioned_reader(k, read_delay=0.30):
    e = r.hgetall(KEY(k))
    if e:
        return e["val"]
    val, ver = db.read(k, delay=read_delay)
    accepted = VERSIONED_SET(keys=[KEY(k)], args=[ver, val, 300])
    print(f"  reader set version {ver} → {'accepted' if accepted else 'REJECTED'}")
    return val

def versioned_writer(k, v):
    ver = db.write(k, v)
    r.delete(KEY(k))
    VERSIONED_SET(keys=[KEY(k)], args=[ver, v, 300])

r.flushdb()
db.write("p4", "old"); r.delete(KEY("p4"))

t = threading.Thread(target=versioned_reader, args=("p4",))
t.start(); time.sleep(0.05)
versioned_writer("p4", "new")        # writes version 2
t.join()                             # reader tries to write version 1 → rejected

print(f"DB={db.data['p4']}  CACHE={r.hget(KEY('p4'), 'val')}")
```

**Result:**
```
  reader set version 1 → REJECTED
DB=new  CACHE=new
```

✅ **Check:** the stale reader was **structurally prevented** from corrupting the cache. No timing guesses, no locks. This is why versioned writes are the strong answer.

---

## Drill 3 — Cache-before-database (phantom data)

```python
def wrong_order(k, v, fail=False):
    r.set(KEY(k), v, ex=300)             # cache FIRST ❌
    if fail:
        raise RuntimeError("db write failed")
    db.write(k, v)

r.flushdb()
db.write("p5", "real")
try:
    wrong_order("p5", "phantom", fail=True)
except RuntimeError:
    pass
print(f"DB={db.data['p5']}  CACHE={r.get(KEY('p5'))}")
```

**Result:** `DB=real  CACHE=phantom`

☠️ **Check:** the cache contains a value that **never existed in the database**. Users see data that isn't real, for 300 seconds. Depending on the domain, this is worse than any staleness.

✅ **Fix:** always write the database first, then touch the cache, and treat the cache operation as best-effort. Re-run with the correct order and a failing database write — the cache is untouched (holds the old value), which is recoverable.

---

## Drill 4 — Caching from a lagging replica

```python
def read_from_replica(k):
    if (v := r.get(KEY(k))) is not None:
        return v
    val, _ = db.read(k, from_replica=True)       # ← replica, not primary
    r.set(KEY(k), val, ex=300)
    return val

r.flushdb()
db.lag = 0.5                                     # 500 ms replication lag
db.write("p6", "old"); time.sleep(0.6)           # let it replicate

db.write("p6", "new")                            # write to primary
r.delete(KEY("p6"))                              # invalidate immediately
time.sleep(0.05)                                 # replica has NOT caught up yet
read_from_replica("p6")                          # ← caches "old" with a fresh TTL
time.sleep(0.6)                                  # replica catches up... too late

print(f"PRIMARY={db.data['p6']}  REPLICA={db.replica['p6']}  CACHE={r.get(KEY('p6'))}")
```

**Result:** `PRIMARY=new  REPLICA=new  CACHE=old`

☠️ **Check:** replication caught up within 500 ms, but the cache captured the pre-replication value and pinned it for 5 minutes. **A transient 500 ms inconsistency became a 5-minute one.**

✅ **Fixes to try:**
1. Read from the **primary** when populating after an invalidation.
2. Delay the invalidation by longer than the replication lag.
3. Versioned writes — the replica returns the old version, which is rejected.

Try fix 3 by combining `versioned_reader` with `from_replica=True`. ✅ **Check:** the stale version is rejected and the cache stays empty until a read sees the new version.

---

## Drill 5 — TTL as the safety net

```python
r.flushdb()
db.write("p7", "old")
r.set(KEY("p7"), "old", ex=3)                    # short TTL

db.write("p7", "new")
# simulate a FAILED invalidation — the DEL never happens (network error, crash)
print(f"immediately : CACHE={r.get(KEY('p7'))}")
time.sleep(3.2)
print(f"after TTL   : CACHE={r.get(KEY('p7'))}")
```

**Result:**
```
immediately : CACHE=old
after TTL   : CACHE=None
```

✅ **Check:** the TTL cleaned up after a completely failed invalidation. Now set `ex=None` and re-run. ✅ **Check:** the value is wrong **forever**.

🎯 **This is why every cached entry must have a TTL**, even when you invalidate diligently. Invalidation is best-effort; the TTL is the only mechanism that catches everything.

---

## Drill 6 — Write-behind data loss

```python
def write_behind_increment(pid):
    r.hincrby("views:pending", pid, 1)            # fast, in memory only

def flush():
    pending = r.hgetall("views:pending")
    for pid, count in pending.items():
        db.write(f"views:{pid}", int(count))
        r.hdel("views:pending", pid)

r.flushdb()
for _ in range(10_000):
    write_behind_increment("p8")
print(f"pending in cache: {r.hget('views:pending', 'p8')}")
print(f"in db           : {db.data.get('views:p8')}")

# simulate a crash before flush
r.delete("views:pending")
flush()
print(f"after crash+flush, db = {db.data.get('views:p8')}")
```

**Result:**
```
pending in cache: 10000
in db           : None
after crash+flush, db = None      ← 10,000 increments lost
```

☠️ **Check:** all 10,000 writes vanished. Now measure what you bought:

```python
import timeit
t_wb = timeit.timeit(lambda: r.hincrby("views:x", "p", 1), number=10_000)
t_wt = timeit.timeit(lambda: db.write("views:y", 1),       number=10_000)
print(f"write-behind: {t_wb:.2f}s   write-through: {t_wt:.2f}s")
print(f"DB writes: 1 (batched) vs 10,000")
```

⚖️ **Check:** you traded 10,000 database writes for 1, and accepted a window of possible loss. **For view counts that's a good trade. For payments it is not.** Now you have both numbers to justify the decision.

---

## Drill 7 — Multi-key invalidation gaps

```python
def update_product(pid, name, category):
    db.write(f"prod:{pid}", name)
    r.delete(KEY(pid))                            # the entity — remembered
    # FORGOT: the rendered fragment and the category listing
r.flushdb()
r.set(KEY("p9"), "Old Kettle", ex=300)
r.set("frag:product:p9", "<div>Old Kettle</div>", ex=300)
r.set("list:category:kitchen", '["Old Kettle"]', ex=300)

update_product("p9", "New Kettle", "kitchen")
print(f"entity   : {r.get(KEY('p9'))}")
print(f"fragment : {r.get('frag:product:p9')}")
print(f"listing  : {r.get('list:category:kitchen')}")
```

**Result:**
```
entity   : None            ✅ invalidated
fragment : <div>Old Kettle</div>    ❌ stale
listing  : ["Old Kettle"]           ❌ stale
```

☠️ **Check:** the user sees a *different* product name depending on which page they're on. This is the most common real-world cache bug — not one wrong value, but **inconsistent values across views of the same data**.

✅ **Fix with version prefixes:**

```python
def update_product_versioned(pid, category):
    db.write(f"prod:{pid}", "New Kettle")
    r.incr(f"ver:prod:{pid}")                     # orphans EVERY key containing that version
    r.incr(f"ver:category:{category}")

def read_product(pid):
    ver = r.get(f"ver:prod:{pid}") or "0"
    return r.get(f"prod:v1:{pid}:{ver}")          # a new version = a guaranteed miss
```

✅ **Check:** one `INCR` invalidates every derived key for that product, in every layer, without enumerating them. This is why version prefixes are the recommended general mechanism.

---

## 📋 Lab checklist

| Drill | Done? | What you proved |
|---|---|---|
| 1 Writer-vs-writer | ☐ | SET races; DEL is commutative |
| 2 Stale set after delete | ☐ | Delete alone isn't enough; versioning is |
| 3 Cache before DB | ☐ | Phantom data is worse than staleness |
| 4 Lagging replica | ☐ | 500 ms lag → 5 min staleness |
| 5 TTL safety net | ☐ | Without a TTL, a lost invalidation is permanent |
| 6 Write-behind | ☐ | 10,000× fewer writes, and real data loss |
| 7 Multi-key gaps | ☐ | Inconsistent views; version prefixes fix it |

**The sentence to remember:** *caches don't go stale for a moment — races make them stale for a TTL. The TTL is the only thing that always saves you.*

---

**← Previous** [3.3.8 The Consistency Problem](08-consistency.md)
**Next →** [3.3.10 Invalidation & Freshness](10-invalidation.md)
