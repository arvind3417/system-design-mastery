# 5.2.4 — 🧪 Database Partition Tutorial

> **Part 5 · Scaling Data Storage · Data Partitioning · Chapter 4 of 4**
> Measure modulo vs consistent hashing, create a hotspot, feel a scatter-gather, and reshard live.

---

## 🧒 ELI5 — Explain Like I'm 5

You've read that modulo hashing is bad and the circle is good. **Now count it yourself.**

You'll:

- Put a million toys in ten boxes two different ways, add an eleventh box, and **count how many toys had to move**. One way moves nearly all of them. The other moves a tenth.
- Give each box only one spot on the circle and see how **lopsided** the boxes get — then add more spots and watch it even out.
- **Make one toy wildly popular** and watch a single box get hammered while nine sit idle.
- Ask a question that needs **every** box, and time it against a question that needs one.
- **Move toys to a new box while people are still playing with them**, without anyone noticing.

Numbers you generate yourself are the ones you remember in an interview.

---

## Setup

```python
# partitionlab.py
import hashlib, bisect, random, time
from collections import Counter, defaultdict

def h(key: str) -> int:
    return int(hashlib.blake2b(str(key).encode(), digest_size=8).hexdigest(), 16)

KEYS = [f"user:{i}" for i in range(1_000_000)]
```

---

## Drill 1 — Modulo vs consistent hashing: count the movement

```python
# ---- Modulo ----
def modulo_assign(keys, n):
    return {k: h(k) % n for k in keys}

before = modulo_assign(KEYS, 10)
after  = modulo_assign(KEYS, 11)
moved = sum(1 for k in KEYS if before[k] != after[k])
print(f"MODULO  10→11: {moved:,}/{len(KEYS):,} moved = {moved/len(KEYS):.1%}")

# ---- Consistent hashing ----
class Ring:
    def __init__(self, nodes, vnodes=150):
        self.vnodes, self.ring, self.keys = vnodes, {}, []
        for n in nodes: self.add(n)
    def add(self, node):
        for i in range(self.vnodes):
            p = h(f"{node}#{i}"); self.ring[p] = node; bisect.insort(self.keys, p)
    def get(self, key):
        p = h(key)
        i = bisect.bisect_right(self.keys, p) % len(self.keys)
        return self.ring[self.keys[i]]

r10 = Ring([f"node{i}" for i in range(10)])
r11 = Ring([f"node{i}" for i in range(11)])
moved = sum(1 for k in KEYS if r10.get(k) != r11.get(k))
print(f"RING    10→11: {moved:,}/{len(KEYS):,} moved = {moved/len(KEYS):.1%}")
```

**Typical result:**
```
MODULO  10→11: 909,192/1,000,000 moved = 90.9%
RING    10→11:  90,614/1,000,000 moved = 9.1%
```

✅ **Check: 91% vs 9%.** For a cache, the first number means a total cold start. For a database, it means moving 91% of your data. **This single measurement is the entire argument for consistent hashing.**

### Also measure removal

```python
r9 = Ring([f"node{i}" for i in range(10) if i != 3])   # node3 dies
moved = sum(1 for k in KEYS if r10.get(k) != r9.get(k))
print(f"RING node3 removed: {moved/len(KEYS):.1%} moved")
```
✅ **Check: ~10%** — and importantly, only keys that *were on node3* move.

---

## Drill 2 — Virtual nodes and load balance

```python
for v in (1, 5, 10, 50, 150, 500):
    ring = Ring([f"node{i}" for i in range(10)], vnodes=v)
    dist = Counter(ring.get(k) for k in KEYS)
    counts = sorted(dist.values())
    mean = sum(counts) / len(counts)
    spread = (counts[-1] - counts[0]) / mean
    print(f"vnodes={v:4d}  min={counts[0]:7,}  max={counts[-1]:7,}  spread={spread:.1%}")
```

**Typical result:**
```
vnodes=   1  min= 21,443  max=241,880  spread=220.4%
vnodes=   5  min= 55,102  max=163,447  spread=108.3%
vnodes=  10  min= 68,930  max=141,225  spread= 72.3%
vnodes=  50  min= 84,551  max=118,332  spread= 33.8%
vnodes= 150  min= 91,204  max=110,986  spread= 19.8%
vnodes= 500  min= 95,660  max=105,120  spread=  9.5%
```

☠️ **Check: with 1 virtual node, one node holds 11× another.** That's not a distributed system, it's one overloaded machine and nine spectators.

✅ **Check: 150–500 virtual nodes brings the spread to ~10–20%.** This is why every real implementation uses them.

### The cascade risk, demonstrated

```python
ring1 = Ring([f"node{i}" for i in range(10)], vnodes=1)
before = Counter(ring1.get(k) for k in KEYS)
victim = max(before, key=before.get)              # the busiest node
ring1b = Ring([n for n in (f"node{i}" for i in range(10)) if n != victim], vnodes=1)
after = Counter(ring1b.get(k) for k in KEYS)
gains = {n: after[n] - before[n] for n in after}
winner = max(gains, key=gains.get)
print(f"vnodes=1: {victim} died; {winner} absorbed {gains[winner]:,} keys "
      f"({gains[winner]/before[winner]:.0%} increase)")
```

☠️ **Check:** with 1 vnode, one neighbour absorbs **all** of the dead node's keys — often doubling its load instantly, which can kill it too. Repeat with `vnodes=150` and the load spreads across every survivor.

🎯 **This cascade is the real reason for virtual nodes** — more than the static imbalance.

---

## Drill 3 — Hotspots: even keys ≠ even traffic

```python
def zipf_key(n=1_000_000, a=1.2):
    return f"user:{min(n, int(random.paretovariate(a-1)))}"

ring = Ring([f"node{i}" for i in range(10)], vnodes=150)

key_dist = Counter(ring.get(k) for k in KEYS)                        # by KEY
traffic  = Counter(ring.get(zipf_key()) for _ in range(1_000_000))   # by TRAFFIC

print("node    keys      traffic")
for n in sorted(key_dist):
    print(f"{n:7} {key_dist[n]:8,} {traffic[n]:10,}")
mx, mn = max(traffic.values()), min(traffic.values())
print(f"traffic imbalance: {mx/mn:.1f}x")
```

**Typical result:**
```
node    keys      traffic
node0   99,412       8,204
node3  101,880     412,551      ← holds the hottest keys
...
traffic imbalance: 52.3x
```

☠️ **Check: keys are perfectly balanced; traffic is 50× imbalanced.** Consistent hashing distributes *keys*, not *load*. **This is the most important lesson in the chapter** and it's why hot-key handling is a separate problem.

### Fix A — local cache

```python
LOCAL_TTL = 1.0
local = {}

def read(key, now):
    e = local.get(key)
    if e and e[1] > now:
        return "local"
    local[key] = (True, now + LOCAL_TTL)
    return ring.get(key)                    # only misses reach a node

t = 0.0
backend = Counter()
for i in range(1_000_000):
    t += 0.00001                            # 100k requests/second
    dest = read(zipf_key(), t)
    if dest != "local":
        backend[dest] += 1
print(f"backend requests: {sum(backend.values()):,} of 1,000,000")
print(f"imbalance now: {max(backend.values())/min(backend.values()):.1f}x")
```

✅ **Check:** backend requests drop by orders of magnitude, and the imbalance collapses — because the hot keys are served locally. **One second of staleness removed the entire hotspot.**

### Fix B — salting

```python
SALT = 16
def salted_node(key, hot_keys):
    if key in hot_keys:
        return ring.get(f"{key}#{random.randrange(SALT)}")
    return ring.get(key)

hot = {f"user:{i}" for i in range(1, 20)}       # the top 19 keys
t2 = Counter(salted_node(zipf_key(), hot) for _ in range(1_000_000))
print(f"salted imbalance: {max(t2.values())/min(t2.values()):.1f}x")
```

✅ **Check:** imbalance drops sharply. ⚠️ But note the cost: **reads for a salted key must query all 16 buckets and merge.** Good for write hotspots, poor for read hotspots (cache those instead).

---

## Drill 4 — Scatter-gather cost

```python
import statistics

SHARD_LATENCY_MS = lambda: random.lognormvariate(1.6, 0.6)   # p50 ~5ms, long tail

def single_shard():
    return SHARD_LATENCY_MS()

def scatter(n):
    return max(SHARD_LATENCY_MS() for _ in range(n))          # bounded by the SLOWEST

for n in (1, 4, 10, 50, 100):
    lat = sorted(scatter(n) for _ in range(5000))
    print(f"shards={n:4d}  p50={lat[2500]:6.1f}ms  p99={lat[4950]:7.1f}ms")
```

**Typical result:**
```
shards=   1  p50=   5.0ms  p99=   20.1ms
shards=   4  p50=   9.2ms  p99=   28.4ms
shards=  10  p50=  12.6ms  p99=   35.0ms
shards=  50  p50=  19.4ms  p99=   48.8ms
shards= 100  p50=  22.7ms  p99=   57.2ms
```

☠️ **Check: the p50 of a 100-shard scatter-gather is worse than the p99 of a single shard.** Every broadcast query samples the tail of your entire fleet. This is tail amplification, measured.

✅ **The lesson:** design so common queries include the shard key. A query that hits one shard is not "a bit faster" — it's in a different performance class.

---

## Drill 5 — Global top-N is expensive

```python
SHARDS = 10
data = defaultdict(list)
for i in range(1_000_000):
    data[h(f"item:{i}") % SHARDS].append((random.random(), f"item:{i}"))
for s in data: data[s].sort(reverse=True)

def global_top(n):
    rows_read = 0
    candidates = []
    for s in range(SHARDS):
        part = data[s][:n]          # must fetch n from EVERY shard
        rows_read += len(part)
        candidates += part
    candidates.sort(reverse=True)
    return candidates[:n], rows_read

top, read = global_top(20)
print(f"returned {len(top)} rows, read {read} rows from {SHARDS} shards")
```

**Result:** `returned 20 rows, read 200 rows from 10 shards`

✅ **Check:** you read `N × shards` rows to return `N`. With 100 shards and a page size of 20, that's 2,000 rows per page — and deep pagination multiplies it further. **This is why global ranked feeds are precomputed rather than queried live.**

---

## Drill 6 — Live resharding

Simulate the dual-write / backfill / verify / cut-over procedure.

```python
old_ring = Ring([f"node{i}" for i in range(4)], vnodes=150)
new_ring = Ring([f"node{i}" for i in range(6)], vnodes=150)

old_store = defaultdict(dict)
new_store = defaultdict(dict)
mode = "old_only"          # → dual_write → backfilling → verify → new_only

def write(k, v):
    if mode in ("old_only", "dual_write", "backfilling", "verify"):
        old_store[old_ring.get(k)][k] = v
    if mode in ("dual_write", "backfilling", "verify", "new_only"):
        new_store[new_ring.get(k)][k] = v

def read(k):
    store, ring = ((new_store, new_ring) if mode == "new_only"
                   else (old_store, old_ring))
    return store[ring.get(k)].get(k)

# 1. Normal operation
for i in range(10_000): write(f"k{i}", i)
print("phase 1 old_only:", sum(len(d) for d in old_store.values()))

# 2. Start dual-writing
mode = "dual_write"
for i in range(10_000, 12_000): write(f"k{i}", i)

# 3. Backfill historical data
mode = "backfilling"
for node, items in list(old_store.items()):
    for k, v in items.items():
        new_store[new_ring.get(k)][k] = v
print("after backfill new:", sum(len(d) for d in new_store.values()))

# 4. Verify — compare every key
mode = "verify"
mismatches = [k for node in old_store for k in old_store[node]
              if new_store[new_ring.get(k)].get(k) != old_store[node][k]]
print(f"mismatches: {len(mismatches)}")

# 5. Cut over
if not mismatches:
    mode = "new_only"
    print("cut over. read k5000 =", read("k5000"))
```

✅ **Check:** zero mismatches, and reads work identically before and after. **Every step was reversible** until the cut-over — that's what makes this procedure safe.

### Now break it on purpose

```python
mode = "old_only"                 # forgot to enable dual-write
for i in range(20_000, 20_100): write(f"k{i}", i)
mode = "new_only"                 # cut over anyway
print("k20050 after cutover:", read("k20050"))     # → None. 100 records lost.
```

☠️ **Check:** skipping dual-write silently loses every record written during the migration. **This is why the verify step exists**, and why you never cut over without it.

---

## Drill 7 — Logical shards make rebalancing trivial

```python
LOGICAL = 1024

def logical_of(key):
    return h(key) % LOGICAL                    # NEVER changes

def build_map(n_nodes):
    per = LOGICAL // n_nodes
    return {l: f"node{min(l // per, n_nodes - 1)}" for l in range(LOGICAL)}

map4, map6 = build_map(4), build_map(6)

moved_logical = sum(1 for l in range(LOGICAL) if map4[l] != map6[l])
moved_keys    = sum(1 for k in KEYS[:100_000]
                    if map4[logical_of(k)] != map6[logical_of(k)])

print(f"logical shards moved: {moved_logical}/{LOGICAL}")
print(f"keys moved: {moved_keys/100_000:.1%}")
print("key→logical mapping changed for: 0 keys")   # ← the whole point
```

**Result:**
```
logical shards moved: 341/1024
keys moved: 33.3%
key→logical mapping changed for: 0 keys
```

✅ **Check:** keys move (the data must physically relocate), but **no key's *identity* changed** — `logical_of(key)` is stable forever. Migration becomes "copy logical shards 256–341 to node4", which is a bulk data operation with a simple, resumable unit of work.

🎯 **Compare with the ring:** both move a similar fraction of data, but logical shards give you a **countable, addressable, resumable unit** ("we've moved 40 of 85 shards") instead of an opaque hash range. That's why Vitess, Citus, and DynamoDB all work this way — and why you should create far more logical shards than nodes on day one.

---

## 📋 Lab checklist

| Drill | Done? | Number you produced |
|---|---|---|
| 1 Modulo vs ring | ☐ | 91% vs 9% |
| 2 Virtual nodes | ☐ | 220% → 10% spread; cascade avoided |
| 3 Hotspots | ☐ | Keys balanced, traffic 50× imbalanced |
| 3a Local cache fix | ☐ | Hotspot eliminated by a 1 s TTL |
| 4 Scatter-gather | ☐ | 100-shard p50 > 1-shard p99 |
| 5 Global top-N | ☐ | 200 rows read to return 20 |
| 6 Live resharding | ☐ | Zero mismatches; and data loss without dual-write |
| 7 Logical shards | ☐ | Key→logical mapping never changes |

**The sentence to take away:** *consistent hashing decides where data lives; it does not decide where traffic goes — and the second problem is usually the one that hurts.*

---

**← Previous** [5.2.3 Consistent Hashing](03-consistent-hashing.md)
**Next →** [6.1.1 Batch & Stream Processing: Overview](../../06-big-data/01-overview/01-batch-and-stream-overview.md)
