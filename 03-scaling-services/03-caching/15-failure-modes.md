# 3.3.15 — Failure Modes

> **Part 3 · Scaling Services · Caching · Chapter 15 of 18**
> The named ways caches break. Learn the names — interviewers use them.

---

## 🧒 ELI5 — Explain Like I'm 5

Four ways the fridge system goes wrong, and they have names:

1. **Everyone's milk runs out at 3pm, so the whole street runs to the shop at 3:01.** One popular thing expired and a hundred people all went to fetch it at once. *(Stampede.)*
2. **Someone keeps asking for "purple milk," which doesn't exist.** It's never in any fridge — because it isn't real — so **every single request goes to the shop.** The fridge protects nothing. If they ask a thousand times a second, the shop dies. *(Penetration.)*
3. **You filled the whole fridge at 2pm, so everything expires at 8pm — all at once.** Now the fridge isn't just missing one thing, it's completely empty, and the whole street needs everything. *(Avalanche.)*
4. **Someone sneaks rotten milk into the fridge with a fresh label on it.** Everyone drinks it and gets sick, and nobody suspects the fridge. *(Poisoning.)*

Four problems. Four fixes: **one person fetches** (locking), **remember that purple milk doesn't exist** (negative caching), **stagger the dates** (jitter), and **check who's allowed to put things in the fridge** (validate what you cache).

---

## 1. Cache stampede (dogpile / thundering herd)

**One hot key expires; every concurrent request misses and hits the origin simultaneously.**

```
t=0.000  key "homepage:feed" TTL expires
t=0.001  1,000 in-flight requests all miss
t=0.002  1,000 identical expensive queries hit the database
t=0.400  the database is saturated; queries slow
t=1.000  callers time out and retry → 2,000 queries
t=∞      collapse
```

☠️ **The severity is proportional to key popularity.** The hottest key — the one caching is helping most — produces the worst stampede.

| Fix | How | Notes |
|---|---|---|
| **Request coalescing / singleflight** | One loader per key; others wait | ✅ The primary fix ([Chapter 5](05-read-patterns.md)) |
| **Distributed lock** | Cross-process coalescing | Needs a TTL and a fallback path |
| **Probabilistic early expiry (xfetch)** | Refresh before expiry, randomly | Lock-free; spreads load |
| **Refresh-ahead** | Background refresh at 80% of TTL | Best for a known small hot set |
| **Serve stale while revalidating** | Return the old value; refresh async | ✅ Users never wait |
| **TTL jitter** | Prevents *synchronised* expiry | Necessary but not sufficient for a single hot key |

---

## 2. Cache penetration

**Requests for keys that don't exist. They always miss, so the cache provides zero protection.**

```
GET /api/products/99999999   → cache miss → DB miss → nothing cached
GET /api/products/99999998   → cache miss → DB miss → nothing cached
... 10,000 QPS of these ...  → 10,000 QPS straight to the database
```

☠️ **This is exploitable.** An attacker requesting random IDs bypasses your cache entirely and hits the origin with every request. It's also caused accidentally by broken crawlers and buggy clients following dead links.

| Fix | How | Trade-off |
|---|---|---|
| **Negative caching** | Cache a null sentinel with a short TTL (15–30 s) | ✅ Simple, effective. Memory for non-existent keys |
| **Bloom filter** | A compact "definitely absent / possibly present" filter in front | ✅ Tiny memory; false positives fall through harmlessly; **cannot delete** from a standard Bloom filter |
| **Input validation** | Reject IDs outside plausible ranges/formats before any lookup | Free; catches the crude cases |
| **Rate limiting per client** | Bounds the damage | Doesn't fix distributed attacks |
| **Authentication on the endpoint** | Removes anonymous abuse | Not always possible |

```python
# Bloom filter guard
if not bloom.might_contain(pid):
    return None                       # certainly absent — never touch the origin
# ... otherwise fall through to cache, then origin
```

⚠️ A Bloom filter has **false positives but no false negatives**: "might contain" may be wrong (you do a wasted lookup), "does not contain" is always right (safe to reject). Use a **counting** or **cuckoo** filter if you need deletions.

---

## 3. Cache avalanche

**A large fraction of the cache expires or disappears at once.**

Two causes:

**(a) Synchronised expiry.** A bulk warm-up or import writes a million keys with identical TTLs at the same moment. They all expire together, forever, in lockstep.

**(b) Cache node/cluster loss.** A restart, a flush, or a failover empties the cache. This is the [cold start](14-cold-start.md) problem.

```
Before:  100,000 QPS × 95% hit → 5,000 QPS to origin
After:   100,000 QPS × 0% hit  → 100,000 QPS to origin (20×)
```

| Fix | Addresses |
|---|---|
| **TTL jitter** (±10–20%) | (a) — spreads expiry over a window |
| **Staggered warm-up** | (a) — don't write everything at once |
| **Replicas + persistence** | (b) — warm recovery instead of cold |
| **Gradual traffic ramp** | (b) — bounded miss rate during recovery |
| **Local caches** | Both — absorb the hottest keys immediately |
| **Circuit breaker + load shedding on the origin** | Both — survive rather than collapse |

---

## 4. Hot key

**One key receives a disproportionate share of traffic, saturating one shard.**

```
100,000 QPS across 10 shards = 10,000 each ✅
One celebrity key = 30,000 QPS → all on ONE shard ❌
```

Aggregate capacity looks fine; one node is on fire. Covered in [Distributed Caching](12-distributed-caching.md); the headline fix is a **local cache with a 1-second TTL**, which cuts load by the number of instances × the TTL.

---

## 5. Big key

**A single very large value degrades everything around it.**

| Problem | Detail |
|---|---|
| Redis is single-threaded | Serialising a 100 MB value **blocks every other client** for that duration |
| Network | 10,000 QPS × 1 MB = 10 GB/s — impossible |
| Migration | `MIGRATE` is blocking per key; a huge key freezes a rebalance |
| Memory spikes | Fragmentation and allocation churn |
| Deletion | `DEL` on a huge collection blocks — use `UNLINK` (async) |

**Fixes:** split into multiple keys (a hash per field, a list in pages), compress, or store the payload in blob storage and cache only the reference (**claim check**).

```bash
redis-cli --bigkeys        # find them before they find you
```

---

## 6. Cache poisoning

**An attacker gets malicious content stored in a shared cache and served to other users.**

| Vector | Example |
|---|---|
| **Unkeyed input** | An `X-Forwarded-Host` header influences the response but isn't in the cache key → the attacker poisons the entry for everyone |
| **Cache deception** | Requesting `/account.css` routes to `/account` but is cached as a "static" file → the attacker then fetches the victim's cached private page |
| **Response splitting** | Injected headers create a second, attacker-controlled response |
| **Caching authenticated content publicly** | One user's data served to all |

**Defences:**
- Everything that influences the response **must** be in the cache key ([Chapter 4](04-cache-key-design.md)).
- Never cache responses that carry `Set-Cookie` in a shared cache.
- Normalise and validate headers at the edge; strip untrusted ones.
- Cache by **file extension only when the origin actually served a static file** — check the response `Content-Type`, not the request path.
- Default to `Cache-Control: private, no-store` for authenticated responses; make public caching opt-in.
- Add a synthetic test: fetch an authenticated page as user A, then as user B, and assert the responses differ.

---

## 7. Cache-as-database

**Treating a cache as durable storage, then discovering it isn't.**

| Symptom | Reality |
|---|---|
| Sessions only in Redis | A failover logs everyone out |
| Rate-limit counters only in Redis | A restart resets every limit — an abuse window |
| The only copy of a computed result in Redis | It's gone; and it may not be recomputable |
| Distributed locks with async replication | A failover can hand the same lock to two holders |

⚠️ **Redis default durability:** `appendfsync everysec` loses up to a second of writes on a crash; replication is asynchronous, so a failover loses whatever hadn't replicated.

**Fixes:** use a durable store as the source of truth; use MemoryDB (durable multi-AZ log) if you truly need Redis semantics with durability; and for locks, use **fencing tokens** so a stale lock holder's writes are rejected by the resource.

---

## 8. Cache pollution

**Low-value data displaces the hot set.** The analytics-scan incident from [Chapter 11](11-eviction-and-sizing.md). Fixes: don't cache scan reads, use frequency-based admission (W-TinyLFU), or give bulk workloads a separate cache.

---

## 9. Cache inconsistency

Covered in depth in [Chapter 8](08-consistency.md). The named failure here is **inconsistent views**: the entity cache is updated but a derived fragment isn't, so the same data appears differently on two pages. Fix with version prefixes or CDC-driven invalidation.

---

## 10. The silent failures

The most dangerous, because monitoring shows green.

| Failure | Why it's invisible |
|---|---|
| **Hit rate silently 0%** | A key-format mismatch between the writer and the reader. Everything works — 10× slower and 10× more expensive. Nobody notices until the bill or a load spike |
| **Cache errors swallowed** | `except: pass` hides a completely dead cache |
| **Stale data with no expiry** | A missed invalidation on a key with no TTL — wrong forever |
| **All responses are fallbacks** | 200 OK, all defaults; dashboards look healthy |
| **Serialisation drift** | New code writes a format old code can't read; a percentage of requests fail per deploy |

**Detection:**
```
cache_hit_rate                     # alert if it drops >10pp below baseline
cache_client_errors_total          # alert if > 0.1%
cache_stale_reads_sampled_total    # 0.1% sampling vs origin ← finds real staleness
fallback_used_total                # alert if the rate is high
cache_keys_by_prefix               # detect a prefix that stopped being written
```

🎯 **The 0.1% sampling check** — on a small fraction of hits, also read the origin and compare — is the only reliable way to detect genuine staleness bugs in production. It costs almost nothing and most teams never do it.

---

## The complete failure-mode table

| # | Failure | Trigger | Primary fix |
|---|---|---|---|
| 1 | Stampede | A hot key expires | Coalescing / singleflight |
| 2 | Penetration | Requests for non-existent keys | Negative caching + Bloom filter |
| 3 | Avalanche | Mass expiry or cache loss | TTL jitter + persistence + ramp |
| 4 | Hot key | Skewed access | Local cache with a short TTL |
| 5 | Big key | Oversized values | Split / compress / claim check |
| 6 | Poisoning | Unkeyed input; caching private data | Complete cache keys; `private` by default |
| 7 | Cache-as-database | Assuming durability | Durable source of truth; fencing tokens |
| 8 | Pollution | Scans through the cache path | Don't cache scans; TinyLFU admission |
| 9 | Inconsistency | Partial invalidation | Version prefixes / CDC |
| 10 | Silent failure | Key mismatch, swallowed errors | Hit-rate alerts + sampling |

**Memorise the names.** An interviewer asking "what can go wrong with your cache?" is looking for exactly this list, and naming stampede, penetration, and avalanche in one breath is a strong signal.

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| Name all ten failure modes | ☐ |
| Distinguish stampede, penetration, and avalanche precisely | ☐ |
| Explain why penetration is exploitable | ☐ |
| Explain Bloom filter semantics (no false negatives) | ☐ |
| Explain why big keys block Redis | ☐ |
| Describe cache deception and its defence | ☐ |
| Explain Redis's actual durability guarantees | ☐ |
| Explain fencing tokens for locks | ☐ |
| Name the five silent failures | ☐ |
| Describe the 0.1% sampling staleness check | ☐ |

---

**← Previous** [3.3.14 Cold Start](14-cold-start.md)
**Next →** [3.3.16 🧪 Lab: The Disaster Drill](16-lab-disaster-drill.md)
