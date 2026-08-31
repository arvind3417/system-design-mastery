# 3.3.11 — Eviction & Sizing

> **Part 3 · Scaling Services · Caching · Chapter 11 of 18**
> Caches are always too small. What you throw away, and how much to buy.

---

## 🧒 ELI5 — Explain Like I'm 5

The fridge is full and you've just bought something new. **Something has to go.** Which?

- **Throw out whatever you haven't touched in the longest time.** Sensible: if you haven't wanted it in three weeks, you probably won't tomorrow. *(LRU — least recently used.)*
- **Throw out whatever you've used the fewest times ever.** Also sensible, and better for things you use rarely but regularly — like the special jam you have every Sunday. LRU would throw the jam out on Saturday; this wouldn't. *(LFU — least frequently used.)*
- **Throw out something at random.** Sounds silly, is surprisingly okay, and costs nothing to work out. *(Random.)*
- **Throw out whatever is closest to its date.** *(TTL-based.)*

And the trap: **one big shopping trip can wipe your whole fridge.** You bring home 50 items you'll never eat again; they push out all your everyday food; now every meal means a trip to the shop. *(Cache pollution from a scan — this is a real and common production incident.)*

Then the money question: **how big should the fridge be?** Bigger is better — but only up to a point. Once it holds everything you eat regularly, a bigger fridge does nothing except cost more.

---

## Eviction policies

| Policy | Evicts | Strengths | Weaknesses |
|---|---|---|---|
| **LRU** (least recently used) | Oldest access | ✅ Good general default; matches temporal locality | Destroyed by scans; ignores frequency |
| **LFU** (least frequently used) | Fewest accesses | ✅ Protects steadily-popular items; scan-resistant | Slow to adapt; old popular items linger |
| **TinyLFU / W-TinyLFU** | Frequency-aware admission + LRU window | ✅ **Best in class** — near-optimal hit rates, scan-resistant | More complex (Caffeine, Redis LFU approximate it) |
| **FIFO** | Oldest inserted | Trivial | Ignores usage entirely |
| **Random** | A random key | Free; surprisingly decent at scale | No locality awareness |
| **TTL / volatile** | Nearest expiry | Predictable | Not usage-aware |
| **ARC** | Adapts between recency and frequency | Excellent | Patent history; less common |
| **Segmented LRU (SLRU)** | Two-tier probation/protected | Scan-resistant LRU | More bookkeeping |

### Redis policies

```
maxmemory 4gb
maxmemory-policy allkeys-lru
```

| Policy | Behaviour |
|---|---|
| `noeviction` | Writes **fail** when full. ✅ Correct when Redis is a datastore, ❌ fatal for a cache |
| `allkeys-lru` | ✅ **The usual choice for a pure cache** |
| `allkeys-lfu` | Better when popularity is stable |
| `allkeys-random` | Cheapest; fine at very high key counts |
| `volatile-lru` | Only evicts keys **with a TTL** — protects version keys and locks |
| `volatile-ttl` | Evicts the nearest-to-expiry first |
| `volatile-random` | Random among keys with a TTL |

⚠️ **Two Redis gotchas worth knowing:**
1. `volatile-*` policies **fail writes** if there are no evictable (TTL-bearing) keys left — the same failure as `noeviction`, arriving unexpectedly.
2. Redis LRU is **approximate**: it samples `maxmemory-samples` keys (default 5) and evicts the best candidate among them. Raising it to 10 improves accuracy at some CPU cost. Redis 4+ LFU is also approximate, using an 8-bit logarithmic counter with time-based decay.

🎯 **Recommendation:** `allkeys-lru` for a pure cache; `volatile-lru` when the same Redis also holds things that must not be evicted (version keys, locks, rate-limit counters) — and give those no TTL so they're protected.

---

## Cache pollution — the scan problem

☠️ **The incident.** A nightly analytics job reads 10 million rows through the same cache-aside code path. Each read caches its result. Within minutes, the entire hot set has been evicted and replaced with rows nobody will read again. At 09:00 the user-facing hit rate is 12% instead of 95%, the database gets 8× its normal load, and everything is slow for an hour while the cache re-warms.

**Defences:**

| Defence | How |
|---|---|
| **Don't cache scan reads** | An explicit `cache=False` on bulk/analytics code paths — the simplest and best fix |
| **Frequency-based admission** (TinyLFU) | A new item is only admitted if it's *more frequent* than the victim it would evict — single-access items never get in |
| **Separate cache namespaces or instances** | Analytics gets its own memory budget |
| **Segmented LRU** | New entries go to a probation segment; they're only promoted on a second access |
| **Route analytics to a replica** | Different path, different (or no) cache |

🎯 **W-TinyLFU's admission filter is the elegant answer** and worth naming: it keeps a compact frequency sketch (Count-Min Sketch) of recent accesses and refuses to admit an item unless its estimated frequency exceeds that of the eviction candidate. A one-off scan therefore cannot displace the hot set at all. Caffeine implements this and it's why Caffeine's hit rates beat plain LRU substantially.

---

## Sizing: how big should the cache be?

### Step 1 — Estimate the working set

$$\text{working set} = (\text{distinct items accessed per window}) \times (\text{avg entry size})$$

```
Distinct products viewed per hour   200,000
Average cached entry                2 KB
Working set                         400 MB
Plus overhead (~1.3x)               520 MB
Round up for headroom               1 GB
```

### Step 2 — Use the Zipf curve, don't guess

Real access follows a power law, so the hit rate against cache size is **steep then flat**:

```
hit rate
 100% ┤                    ╭──────────────
      │              ╭─────╯
  80% ┤        ╭─────╯
      │    ╭───╯
  50% ┤ ╭──╯
      │╭╯
   0% ┼────────────────────────────────────
      0   1%   5%   20%   50%  100% of dataset cached
             ↑ the knee
```

🔢 Typical for Zipf(α≈1): caching **1%** of the dataset gives ~50% hit rate; **10%** gives ~85%; **20%** gives ~92%; doubling to 40% gives ~95%. **The marginal return collapses after the knee.**

**Method:** measure it. Run the cache with `maxmemory` at several sizes and plot hit rate ([Read Drill, Drill 8](06-lab-read-drill.md#drill-8--hit-rate-vs-cache-size)). Size at the knee plus ~30%.

### Step 3 — Check it against your budget

$$\text{required hit rate} = 1 - \frac{\text{origin capacity}}{\text{total QPS}}$$

```
Total read QPS         50,000
Database capacity      10,000 QPS
Required hit rate      1 - 10,000/50,000 = 80%
```

If your measured curve says 80% needs 4 GB, buy 6 GB. If 80% needs 400 GB, the cache is not the right solution — you need a different data model, a CDN, or precomputation.

### Step 4 — Account for overhead

| Overhead | Factor |
|---|---|
| Redis per-key overhead (key string, dict entry, expiry, robj) | ~50–100 bytes per key |
| Serialisation expansion (JSON) | 1.5–3× vs compact binary |
| Replication (a replica holds a full copy) | ×2 for a replicated cluster |
| Fragmentation | 1.1–1.5× |
| Headroom before eviction thrash | ÷0.7 |

🔢 **10 million keys × 100 bytes of overhead = 1 GB of pure metadata**, before any values. At small value sizes, overhead can exceed data — which is an argument for storing hashes (`HSET`) of related fields rather than thousands of tiny individual keys.

---

## Value size matters

| Entry size | Consideration |
|---|---|
| < 100 B | Per-key overhead dominates — group into hashes |
| 100 B – 10 KB | ✅ The sweet spot |
| 10 KB – 100 KB | Watch network bandwidth: 10,000 QPS × 50 KB = 500 MB/s |
| 100 KB – 1 MB | Consider compression; check serialisation CPU |
| > 1 MB | ⚠️ Redis blocks on large values; store in blob storage and cache the key |

**Compression** (LZ4, Snappy, zstd) typically gives 3–10× on JSON at a small CPU cost. Worth it above ~10 KB; usually a net loss below ~1 KB.

⚠️ **Redis is single-threaded for command execution.** A 10 MB value takes milliseconds to serialise and transfer, blocking *every other client* for that time. Large values are a latency hazard, not just a memory one.

---

## Memory management in Redis

```bash
INFO memory
# used_memory              actual data
# used_memory_rss          what the OS sees
# mem_fragmentation_ratio  rss / used — want ~1.0-1.5
# maxmemory                the cap
# evicted_keys             cumulative evictions ← the key metric
# keyspace_hits / misses   hit rate
```

| Symptom | Meaning | Action |
|---|---|---|
| `evicted_keys` rising steadily | Cache too small — you're thrashing | Add memory, or reduce what you cache |
| `evicted_keys` = 0 and memory near max | At capacity but stable | Fine, but no headroom |
| `mem_fragmentation_ratio > 1.5` | Fragmentation | Enable `activedefrag`, or restart |
| `mem_fragmentation_ratio < 1.0` | **Swapping** | ⚠️ Serious — Redis performance collapses. Fix immediately |
| Hit rate falling with stable traffic | Working set grew, or pollution | Investigate key growth by prefix |

**`used_memory` vs `maxmemory`:** set `maxmemory` to about **70–75% of the instance's RAM**. Redis needs headroom for replication buffers, client output buffers, and copy-on-write during `BGSAVE` — a fork can transiently double memory usage on a write-heavy instance.

---

## What to cache, given limited memory

Rank candidates by value per byte:

$$\text{value} = \frac{(\text{access frequency}) \times (\text{cost to recompute})}{\text{size in bytes}}$$

| Candidate | Frequency | Recompute cost | Size | Verdict |
|---|---|---|---|---|
| Hot product entity | Very high | Low (indexed read) | 2 KB | ✅ Cache |
| Expensive aggregate report | Low | Very high (10 s query) | 50 KB | ✅ Cache — the cost dominates |
| A user's full order history | Low | Medium | 500 KB | ❌ Too big for its frequency |
| Rendered homepage | Very high | High | 200 KB | ✅ Cache (or better, at the CDN) |
| A single user's session | High for one user | Low | 1 KB | ✅ Cache — required for statelessness |
| Analytics scan results | Once | High | Huge | ❌ Never cache |

🎯 **Note the second row:** infrequently-accessed data is still worth caching if it's very expensive to produce. Frequency alone is not the criterion.

---

## Sizing worked example

```
System: e-commerce product catalogue
  Catalogue                 5,000,000 products
  Entry size                3 KB (JSON), 1 KB compressed
  Peak read QPS             40,000
  Database capacity         8,000 QPS
  Required hit rate         1 - 8,000/40,000 = 80%

From the measured Zipf curve, 80% hit rate needs ~8% of the catalogue:
  400,000 products × 1 KB (compressed)      = 400 MB
  + per-key overhead 400,000 × 100 B        = 40 MB
  + fragmentation ×1.2                      = 528 MB
  + headroom ÷0.7                           = 754 MB
  → provision 1 GB of usable cache
  → maxmemory 1 GB on instances with ~1.5 GB RAM
  → with a replica: 2 × 1.5 GB instances

Cost check: ~3 GB of managed Redis ≈ $100-150/month
Alternative: 5 extra database read replicas ≈ $2,000+/month
→ The cache is 15x cheaper for this workload.
```

That final cost comparison is exactly the kind of reasoning that turns "add a cache" into a defended decision.

---

## Multi-tier sizing

| Tier | Size | Rationale |
|---|---|---|
| Local in-process | 10–100 MB per instance | Only the very hottest keys; short TTL; limited by heap and GC pressure |
| Redis | GBs | The main working set |
| CDN | Provider-managed | Effectively unlimited for public content |

**Don't oversize the local cache.** It competes with your application heap, increases GC pause times (which hurts p99 — the thing you were trying to fix), and holds N redundant copies across the fleet. A few thousand entries with a 1–10 s TTL captures nearly all the benefit.

---

## ☠️ Failure modes

| Mistake | Consequence |
|---|---|
| `noeviction` on a cache | Writes fail when full; the cache stops accepting new data |
| `volatile-*` with no TTL'd keys | Writes fail unexpectedly |
| Undersized cache | Constant eviction thrash; low hit rate; the memory is wasted anyway |
| Oversized cache | Money spent past the knee for no benefit |
| Analytics sharing the cache | Pollution wipes the hot set |
| `maxmemory` set to 100% of RAM | OOM during `BGSAVE` fork or a replication burst |
| Millions of tiny keys | Per-key overhead exceeds the data |
| Very large values | Redis blocks; tail latency spikes for all clients |
| No eviction monitoring | Silent hit-rate degradation as data grows |
| Oversized local caches | GC pauses worsen the p99 you were optimising |

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| Compare LRU, LFU, TinyLFU, random | ☐ |
| Choose a Redis `maxmemory-policy` and justify it | ☐ |
| Explain the two `volatile-*` gotchas | ☐ |
| Narrate the analytics-scan pollution incident and five defences | ☐ |
| Explain W-TinyLFU's admission filter | ☐ |
| Size a cache from working set, curve, and required hit rate | ☐ |
| Derive required hit rate from origin capacity | ☐ |
| Account for all five overhead factors | ☐ |
| Explain why `maxmemory` should be ~70% of RAM | ☐ |
| Interpret `evicted_keys` and `mem_fragmentation_ratio` | ☐ |
| Rank cache candidates by frequency × cost ÷ size | ☐ |

---

**← Previous** [3.3.10 Invalidation & Freshness](10-invalidation.md)
**Next →** [3.3.12 Distributed Caching](12-distributed-caching.md)
