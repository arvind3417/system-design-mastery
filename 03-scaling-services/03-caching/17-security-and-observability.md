# 3.3.17 — Security & Observability

> **Part 3 · Scaling Services · Caching · Chapter 17 of 18**
> Caches hold your most-accessed data in the least-protected place, and fail in ways that look like success.

---

## 🧒 ELI5 — Explain Like I'm 5

Two things nobody thinks about until it's too late.

**Security.** The fridge is in a shared hallway. Anyone can open it. Two problems follow:

- **Whose milk is whose?** If the bottles aren't labelled with names, Ann might grab Bob's, and Bob's bottle has his diary taped to it. *(Cache keys must include who the data belongs to.)*
- **Can anyone put things in?** If a stranger can add a bottle labelled "Ann's milk," Ann will drink whatever they put there. *(Only trusted code should write to the cache.)*

**Observability.** The fridge doesn't tell you when it stops working. It doesn't beep. It just quietly stops being cold, and you keep putting food in, and you don't find out until someone gets ill.

So you put a **thermometer** in it. And you occasionally **taste-test** something and compare it against a fresh one from the shop, to check the fridge isn't lying to you.

Caches fail *silently and successfully* — 200 OK, every time, with the wrong data. Only measurement finds that.

---

## Part 1 — Security

### 1. Key scoping is access control

☠️ **The most common cache security bug**: a shared key for per-user data.

```python
# BROKEN — invoice 123 is cached globally
cache.set(f"invoice:{invoice_id}", data)
# Any tenant that can name invoice 123 gets it, bypassing the database's own check
```

```python
# CORRECT
authorize(user, invoice_id)                             # ← always, before the cache
cache.set(f"invoice:v2:{user.tenant_id}:{invoice_id}", data, ex=300)
```

**Two rules, both mandatory:**
1. **Authorize before consulting the cache** — never only on the miss path. A cache hit must not skip the check.
2. **Every dimension that affects visibility goes in the key** — tenant, user, role, permission scope.

⚠️ **Role in the key:** if an admin sees extra fields, `product:9` cached by an admin and served to a normal user leaks those fields. Either include the role in the key, or cache the *maximal* object and filter fields at render time (better — one cache entry, filtering per request).

### 2. What must never be cached

| Data | Why |
|---|---|
| Passwords, password hashes | No reason to; enormous downside |
| Full payment card numbers, CVV | PCI-DSS violation |
| Session tokens in a *shared, unscoped* key | Session hijacking |
| Private keys, API secrets | Use a secrets manager |
| Data the user has requested be deleted | GDPR — see below |
| Anything where staleness creates a security hole | Revoked permissions, disabled accounts |

🎯 **The revocation case is the subtle one.** Caching a permission decision for 5 minutes means a fired employee retains access for 5 minutes after revocation. Sometimes acceptable, sometimes not — **decide deliberately and write it down**. Permission and feature-flag TTLs should be seconds, not minutes.

### 3. GDPR and the right to erasure

A deletion request must reach **every** cache layer:

```python
def delete_user_data(uid):
    db.delete_user(uid)
    cache.delete(*cache.smembers(f"tag:user:{uid}"))   # tagged keys
    cache.incr(f"ver:user:{uid}")                       # orphan anything versioned
    local_cache_invalidate_broadcast(uid)               # pub/sub to instances
    cdn.purge_by_tag(f"user-{uid}")                     # the edge
    # Browser caches: cannot be purged → rely on short max-age for personal data
```

⚠️ **Design for this in advance.** If personal data is spread across untagged keys, honouring a deletion request means flushing the entire cache. **Tag or version every key that contains personal data** from the start.

### 4. Transport and at-rest protection

| Control | Why |
|---|---|
| **TLS to the cache** (`rediss://`) | Cache traffic is plaintext by default and contains your hottest data |
| **AUTH / ACLs** | Redis is unauthenticated by default. Internet-exposed unauthenticated Redis is a well-known compromise vector |
| **Network isolation** | Private subnet, security groups, no public IP. Non-negotiable |
| **Per-service ACL users** | `ACL SETUSER svc-orders on >pw ~orders:* +get +set` — least privilege by key pattern |
| **Disable dangerous commands** | `rename-command FLUSHALL ""`, `KEYS ""`, `CONFIG "..."`, `DEBUG ""` |
| **Encryption at rest** | For snapshot files if the cache holds sensitive data |

```
# Least-privilege ACL: this service can only touch its own key prefix
ACL SETUSER orders-svc on >secret ~orders:* ~ver:orders:* +@read +@write -@dangerous
```

### 5. Cache poisoning and deception

Covered in [Failure Modes](15-failure-modes.md). The security-relevant summary:

| Attack | Defence |
|---|---|
| **Unkeyed input** — a header influences the response but isn't in the key | Everything affecting the response goes in the key; strip untrusted headers at the edge |
| **Cache deception** — `/account.css` routes to `/account` and is cached as static | Cache by the **response** `Content-Type`, not the request path extension |
| **Caching authenticated responses publicly** | `private, no-store` by default; never cache responses carrying `Set-Cookie` |
| **Response splitting** | Sanitise anything reflected into headers |

**Add a synthetic test to CI:** fetch an authenticated page as user A, then as user B, and assert the responses differ. It catches the whole class.

### 6. Denial of service via the cache

| Vector | Defence |
|---|---|
| **Penetration** — random non-existent IDs bypass the cache | Negative caching, Bloom filter, input validation |
| **Key-space flooding** — unbounded query parameters create unlimited keys, evicting the hot set | Parameter allowlist; normalise before keying |
| **Big-value insertion** — a huge cached response blocks the single-threaded cache | Cap cached value size; reject oversized responses |
| **Expensive-key targeting** — repeatedly requesting the most costly uncached computation | Rate limit; precompute; coalesce |

---

## Part 2 — Observability

### The metrics that matter

```
# Effectiveness
cache_hits_total{layer, key_prefix}
cache_misses_total{layer, key_prefix}
cache_hit_ratio{layer, key_prefix}              # ← the headline SLI

# Health
cache_operation_duration_seconds{op}            # p50/p99 — should be ~1 ms
cache_errors_total{type}                        # timeout | connection | serialization
cache_circuit_breaker_state

# Capacity
cache_memory_used_bytes / cache_memory_max_bytes
cache_evicted_keys_total                        # rising = too small
cache_keys_total{prefix}                        # detect a prefix that stopped being used
cache_key_size_bytes                            # histogram — find big keys

# Correctness
cache_invalidations_total{reason}
cache_invalidation_failures_total               # ← should be ~0
cache_stale_reads_sampled_total                 # the 0.1% comparison check

# Impact — the ones that actually matter
origin_qps                                      # what the cache is protecting
origin_qps_saved_estimate                       # hits × cost per origin call
```

🎯 **Label by `key_prefix`.** A global hit rate of 90% can hide `product:*` at 99% and `feed:*` at 5%. Per-prefix rates are how you find the cache that isn't working.

### The alerts

| Alert | Condition | Severity |
|---|---|---|
| Hit rate collapse | > 10 pp below the 7-day baseline for 5 min | ⚠️ Page — usually a key-format bug or a cold start |
| Cache errors | > 1% of operations for 2 min | ⚠️ Page |
| Cache p99 latency | > 10 ms for 5 min | Warn |
| Evictions rising | Sustained increase over baseline | Warn — the working set has grown |
| Memory > 90% of max | | Warn |
| Invalidation failures | > 0 | Warn |
| **Origin QPS > 70% of capacity** | | ⚠️ Page — the leading indicator |
| Fallback rate high | > 5% | Warn |
| Replication link down | Any | ⚠️ Page |

**Alert on symptoms (hit rate, origin load), not causes (memory, CPU).** A cache at 95% memory that is happily evicting cold keys is *working correctly*. A cache at 20% memory with a 0% hit rate is broken.

### The staleness sampler — the technique nobody uses

Most caching bugs are **wrong data**, not slow data, and no infrastructure metric detects them.

```python
def get_with_sampling(pid):
    val = cache_get(pid)
    if random.random() < 0.001:                      # 0.1% of hits
        background.submit(compare_with_origin, pid, val)
    return val

def compare_with_origin(pid, cached):
    fresh = db.get_product(pid)
    if normalize(fresh) != normalize(cached):
        metrics.increment("cache.stale_detected", tags={"prefix": "product"})
        logger.warning("stale cache", extra={"key": pid,
                                             "cached": cached, "fresh": fresh})
```

🎯 **This is the only reliable way to find real staleness bugs in production.** It costs 0.1% extra origin load and it catches missed invalidations, key collisions, and serialisation drift — the bugs that otherwise surface as customer complaints months later.

### Distributed tracing

Add spans for cache operations so a trace shows exactly where time went:

```
span: GET /api/products/9                      145 ms
  ├─ span: cache.get product:v3:9:en:GBP        0.8 ms   [miss]
  ├─ span: db.query products                  138 ms
  └─ span: cache.set product:v3:9:en:GBP        0.9 ms
```

Tag spans with `cache.hit=true|false` and you can compute hit rate *per endpoint* from traces, and immediately see which endpoints are cache-ineffective.

### Debugging tools

```bash
redis-cli INFO stats                 # keyspace_hits / keyspace_misses
redis-cli INFO memory                # used, fragmentation, evictions
redis-cli --bigkeys                  # find oversized values
redis-cli --hotkeys                  # requires maxmemory-policy allkeys-lfu
redis-cli --latency-history          # latency over time
redis-cli SLOWLOG GET 25             # the slowest commands
redis-cli --scan --pattern 'prod:*' | head    # ⚠️ SCAN, never KEYS
redis-cli MEMORY USAGE mykey         # size of one key
redis-cli OBJECT FREQ mykey          # LFU access counter
```

☠️ **Never run `KEYS` or `MONITOR` on a production cache.** `KEYS` is O(N) and blocks the single-threaded server. `MONITOR` streams every command and can consume most of the server's throughput. Use `SCAN` and brief `SLOWLOG` samples instead.

### Cost observability

```
cache_cost_monthly            = provisioned GB × $/GB-month
origin_cost_avoided_estimate  = hits × (cost per origin query)
cache_roi                     = avoided / cost
```

If ROI < 1, the cache is costing more than it saves — either the hit rate is too low (fix the keys or the TTL) or the cached data is too cheap to compute (stop caching it). **Very few teams measure this**, and it's a compelling thing to mention.

---

## The caching dashboard

```
┌─────────────────────────────────────────────────────────┐
│ HIT RATE           93.2%  ▁▂▃▅▇█▇▅▃  (7d baseline 94.1%)│
│ ORIGIN QPS         4,820 / 12,000 capacity (40%)        │
│ CACHE p99          1.2 ms                               │
│ ERRORS             0.02%                                │
├─────────────────────────────────────────────────────────┤
│ BY PREFIX      hit%    keys      mem     evict/s        │
│  product:      99.1%   412k      820MB   0              │
│  feed:         87.4%   180k      340MB   12             │
│  session:      99.8%    90k       90MB   0              │
│  search:       61.2%    50k      110MB   240   ← low    │
├─────────────────────────────────────────────────────────┤
│ MEMORY  1.36 / 2.00 GB   FRAG 1.12   EVICTED 4.2k/h     │
│ STALE DETECTED (sampled)  0 in 24h                      │
└─────────────────────────────────────────────────────────┘
```

Everything an on-call engineer needs in one screen: is it working, is the origin safe, and which prefix is the problem.

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| State the two mandatory key-scoping security rules | ☐ |
| Explain the role-in-key problem and the better alternative | ☐ |
| List what must never be cached | ☐ |
| Explain the permission-revocation staleness risk | ☐ |
| Design cache deletion for GDPR across all layers | ☐ |
| Configure Redis ACLs with least privilege | ☐ |
| Name the commands to disable in production | ☐ |
| Name four cache-based DoS vectors | ☐ |
| List the metrics grouped by effectiveness / health / capacity / correctness | ☐ |
| Explain why you label by `key_prefix` | ☐ |
| Implement the 0.1% staleness sampler | ☐ |
| Explain why `KEYS` and `MONITOR` are forbidden | ☐ |
| Compute cache ROI | ☐ |

---

**← Previous** [3.3.16 🧪 Lab: The Disaster Drill](16-lab-disaster-drill.md)
**Next →** [3.3.18 Interview Walkthrough](18-interview-walkthrough.md)
