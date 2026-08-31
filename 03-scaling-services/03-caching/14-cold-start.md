# 3.3.14 — Cold Start

> **Part 3 · Scaling Services · Caching · Chapter 14 of 18**
> The empty cache. Every deploy, every restart, every scale-out, every failover.

---

## 🧒 ELI5 — Explain Like I'm 5

You just moved into a new house. **The fridge is empty.**

For the next few days, *every single meal* means a trip to the shop. It's exhausting — and if the whole street moved in on the same day, the shop would be overwhelmed.

Eventually the fridge fills up with the things you actually eat, and life returns to normal. **The problem is the getting-there, not the destination.**

So what do you do?

- **Bring some food with you** when you move in. *(Snapshot restore, or copying from the old fridge.)*
- **Don't invite the whole street to move in on the same day.** Move one house at a time. *(Gradual traffic ramp, rolling deploys.)*
- **Buy the ten things you know you'll want** before you need them. *(Pre-warming the known hot set.)*
- **Ask the shop first** whether it can handle everyone shopping at once. If it can't, **let people in a few at a time.** *(Check origin capacity; shed or queue.)*

The mistake is treating an empty fridge as a temporary inconvenience. **At scale it is an outage.**

---

## When cold starts happen

| Event | Scope | Frequency |
|---|---|---|
| Cache node restart or upgrade | One shard | Weekly/monthly |
| Cache cluster failover | One shard | Occasional |
| **Full cache flush** (accidental or deliberate) | Everything | Rare, severe |
| **Deploying a new key format / version bump** | Everything | Every relevant deploy |
| Application scale-out (new instances) | Local caches only | Continuously |
| Application deploy | All local caches | Daily |
| New region brought online | That region | Rare |
| CDN purge-all | The edge | Rare, severe |
| Traffic failover to another region | The receiving region | Rare |
| Cache eviction storm (working set grew) | Partial, ongoing | Creeping |

☠️ **The key-version bump is the sneaky one.** Changing `prod:v3:` to `prod:v4:` is a *correct* and *recommended* practice ([Chapter 4](04-cache-key-design.md)) — and it invalidates 100% of the cache instantly. **If you don't ramp the deploy, you have deliberately caused a total cold start at your busiest hour.**

---

## The maths of a cold start

```
Steady state:  100,000 QPS × 5% miss  =   5,000 QPS to origin
Cold:          100,000 QPS × 100% miss = 100,000 QPS to origin   (20×)
Origin capacity: 12,000 QPS
→ The origin is at 8x its capacity. It does not "get slow" — it stops.
```

**Recovery is not automatic.** The origin can only serve 12,000 QPS, so the cache fills at 12,000 keys/sec at best — while the remaining 88,000 QPS time out, retry, and add more load. Without intervention the system may **never** recover, because filling the cache requires origin capacity that the miss storm is consuming.

🎯 **This is the crucial insight: a cold cache is not a transient slow period, it is a stable failure state.** Escaping it requires actively *reducing* demand (shedding) or *increasing* supply (pre-warming). Saying this in an interview demonstrates you've thought past the happy path.

### How long does warming take?

$$t_{\text{warm}} \approx \frac{\text{keys needed for target hit rate}}{\text{origin capacity available for misses}}$$

```
Keys needed for 90% hit rate     400,000
Origin spare capacity            7,000 QPS (12,000 total − 5,000 baseline)
Warm time                        ≈ 57 seconds  (if demand is controlled)
```

But at full traffic with no shedding, spare capacity is **negative** — hence the stable failure.

---

## Prevention strategies

### 1. Snapshot persistence (cheapest win)

```
# redis.conf
save 900 1
save 300 10
save 60 10000
```

A restart reloads the RDB and comes back **warm**. This turns "restart = cold start" into "restart = a few seconds of downtime with the data intact."

⚖️ Costs: fork-and-copy memory spikes during `BGSAVE` (keep `maxmemory` at ~70% of RAM), and disk I/O. Worth it for any cache large enough that a cold start is dangerous.

### 2. Failover to a warm replica

A replica has been receiving the full replication stream, so it is already warm. **Promoting a replica is a warm recovery; starting a new node is a cold one.** This is one of the strongest arguments for cache replicas, separate from durability.

### 3. Gradual traffic ramp

Never send 100% of traffic at a cold cache.

```
new cache: 1% → 5% → 20% → 50% → 100%, waiting for the hit rate
           to stabilise at each step
```

The cache warms progressively from a manageable miss rate at each level. This is the standard procedure for cache migrations and cluster replacements.

### 4. Pre-warming

```python
def prewarm():
    # Load the known hot set before accepting traffic
    hot_ids = analytics.top_n_products(n=100_000, window="24h")
    for batch in chunks(hot_ids, 500):
        rows = db.query("SELECT * FROM products WHERE id = ANY(%s)", batch)
        pipe = cache.pipeline()
        for row in rows:
            pipe.set(product_key(row.id), serialize(row),
                     ex=int(300 * random.uniform(0.9, 1.1)))   # jitter!
        pipe.execute()
        time.sleep(0.05)                 # ← throttle: don't DoS your own database
```

⚠️ Two details matter: **throttle** the warm-up so it doesn't itself overload the origin, and **jitter the TTLs** so the pre-warmed keys don't all expire together five minutes later and cause a second storm.

**Where do hot IDs come from?** Analytics (top-N over the last day), the previous cache's key dump, or an access log sample. Even a rough list captures most of the benefit, because of the Zipf skew.

### 5. Shadow / mirrored traffic

Send a copy of production reads to the new cache tier before cutting over. It warms on real traffic with real distribution — the highest-fidelity approach, and the standard for zero-downtime cache migrations.

### 6. Dual-read during migration

```python
def get(key):
    if (v := new_cache.get(key)) is not None:
        return v
    if (v := old_cache.get(key)) is not None:
        new_cache.set(key, v, ex=300)          # copy forward on read
        return v
    return load_from_origin(key)
```
The new cache warms from the old one at zero origin cost. Elegant, and the right way to migrate between cache clusters.

### 7. Local caches as shock absorbers

Even with a completely cold distributed cache, per-instance local caches with a 1–5 second TTL immediately reduce origin load on hot keys by orders of magnitude. **They warm in milliseconds** because they only hold a few hundred keys.

---

## Deploy-time cold starts

**Application deploys flush every local cache**, because processes restart.

| Mitigation | How |
|---|---|
| **Rolling deploys** | Only a fraction of instances are cold at once — the rest still serve from their local caches |
| Keep local caches small and short-TTL | They re-warm in seconds; the impact is brief |
| Warm on startup before readiness passes | The instance doesn't receive traffic until warm |
| Don't bump the shared key version and deploy at peak simultaneously | Compounding cold starts |

⚠️ **Slow startup blocks autoscaling.** If warming takes 60 seconds before readiness, your autoscaling response time is 60 seconds longer, exactly when you need capacity. Keep startup warming to the essential minimum, or warm lazily in the background while serving.

---

## Detecting and responding to a cold start

```
cache_hit_rate                  → drops to ~0
origin_qps                      → spikes 10-20x
origin_latency_p99              → climbs
cache_keyspace_size             → near zero, climbing
cache_fill_rate_keys_per_second → how fast you're recovering
```

**Automatic response:**

```python
if cache.hit_rate() < 0.5 and origin.load() > 0.8:
    load_shedder.enable(priority_below=NORMAL)   # protect the origin
    coalescer.enable_aggressive()                # one origin call per key, globally
    logger.warn("cold start detected, shedding enabled")
```

🎯 **Automatic shedding on detected cold start** is what separates a 2-minute blip from a 2-hour outage. Manual response is too slow: by the time a human is paged, the origin is already saturated.

---

## The full-flush incident

☠️ **A real and recurring class of incident:** someone runs `FLUSHALL` — on the wrong cluster, via a misconfigured tool, or as part of a "let's clear the cache to fix the stale data" reflex during an unrelated incident.

**Consequences:** instant 100% miss rate at full production traffic. Almost always an outage.

**Prevention:**
```
rename-command FLUSHALL ""          # disable it entirely
rename-command FLUSHDB  ""
rename-command KEYS     ""          # also blocks the whole server
rename-command CONFIG   "CONFIG_a7f3c1"
```
Plus: separate credentials for production, no interactive `redis-cli` access to production from laptops, and a documented procedure that says **"clearing the cache during an incident may cause a bigger incident."**

**If it happens:** shed immediately, ramp traffic back, and pre-warm from analytics rather than letting organic traffic do it.

---

## Cold start in other layers

| Layer | Cold start cause | Mitigation |
|---|---|---|
| **CDN** | Purge-all, or a new PoP | Purge by surrogate key, never everything; origin shielding limits the storm to one request per object |
| **Database buffer pool** | Restart or failover | `pg_prewarm`, or a warm standby that's been serving reads |
| **JIT / runtime** | Every process start | Warm-up requests before readiness |
| **Connection pools** | Every process start | Pre-establish minimum idle connections |
| **ML model cache** | Deploy | Load and run a warm-up inference at startup |
| **Serverless** | Every cold invocation | Provisioned concurrency |

**Origin shielding at the CDN deserves repeating here:** without it, a purge-all causes one origin request *per PoP per object* — potentially 300× amplification. With it, one.

---

## The design question to ask

> **"Can the origin serve 100% of traffic, even briefly?"**

| Answer | Implication |
|---|---|
| **Yes** | The cache is a genuine optimisation. Cold starts are a latency event, not an availability one. |
| **No** | The cache is a **critical dependency**. You must have: warm failover, snapshot persistence, gradual ramping, local caches, coalescing, and automatic shedding. |

Most systems at scale are in the second category and haven't admitted it. 🎯 **Stating this explicitly — "our cache is load-bearing, so here's the cold-start plan" — is a genuinely senior thing to say.**

---

## ☠️ Failure modes

| Mistake | Consequence |
|---|---|
| No persistence and no replicas | Every restart is a cold start |
| Restoring full traffic to a cold cache | The recovery causes a second outage |
| Key version bump deployed at peak | Deliberate total cold start at the worst moment |
| Pre-warming without throttling | The warm-up overloads the origin |
| Pre-warming without TTL jitter | A synchronised expiry storm minutes later |
| `FLUSHALL` accessible in production | One command from an outage |
| No local caches | Nothing absorbs the storm |
| No automatic shedding | Humans respond too slowly |
| Long warm-up before readiness | Autoscaling can't respond in time |
| CDN purge-all without shielding | 300× origin amplification |

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| List ten events that cause a cold start | ☐ |
| Explain why a cold cache is a *stable* failure state, not a transient one | ☐ |
| Compute warm time from origin spare capacity | ☐ |
| Name seven prevention strategies | ☐ |
| Explain why replica promotion is warm and node replacement is cold | ☐ |
| Write a throttled, jittered pre-warm routine | ☐ |
| Describe dual-read cache migration | ☐ |
| Explain deploy-time local cache cold starts and rolling deploys | ☐ |
| Configure Redis to disable `FLUSHALL` and `KEYS` | ☐ |
| Answer "can the origin serve 100% of traffic?" for your design | ☐ |

---

**← Previous** [3.3.13 Cache High Availability](13-cache-high-availability.md)
**Next →** [3.3.15 Failure Modes](15-failure-modes.md)
