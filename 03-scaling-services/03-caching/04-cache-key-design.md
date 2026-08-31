# 3.3.4 — Cache Key Design

> **Part 3 · Scaling Services · Caching · Chapter 4 of 18**
> The key determines your hit rate, your invalidation strategy, and whether you leak one user's data to another.

---

## 🧒 ELI5 — Explain Like I'm 5

A cache is a wall of labelled drawers. The **label** is the key.

Get the labels wrong and everything falls apart:

- **Label too vague** — you write `"drink"` on a drawer. Now Ann's coffee and Bob's tea both want that drawer, and **whoever opens it gets whatever the last person put in.** Ann gets Bob's tea. *(That's a cache collision — and if the drawer held Bob's bank balance, it's a data leak.)*
- **Label too specific** — you write `"coffee, Tuesday, 3:04pm, slightly cloudy, Ann was wearing red"`. Nobody will ever ask for exactly that again, so **every drawer is used once and then wasted.** *(Hit rate near zero.)*
- **Label unpredictable** — sometimes you write `"Coffee"`, sometimes `"coffee"`, sometimes `"COFFEE "` with a space. Now you have three drawers for the same thing, and you can never find the one you want. *(Inconsistent key construction.)*
- **Label can't be found later** — you need to throw out everything about Ann, but the labels don't mention Ann in a findable way, so you'd have to open all million drawers. *(Unbounded invalidation.)*

Good labels are **exactly specific enough**, **built the same way every time**, and **easy to find in groups**.

---

## What a key must encode

**Everything that changes the value.** Miss one thing and you serve the wrong data.

```
product:v3:9:GBP:en-GB
│       │  │ │   └ locale         — text differs
│       │  │ └ currency           — price differs
│       │  └ entity id
│       └ schema version          — lets you roll forward
└ namespace / entity type
```

The test: *"if I changed this input, would the response differ?"* If yes, it belongs in the key.

| Dimension | Include when |
|---|---|
| Entity type + ID | Always |
| **User / tenant ID** | The response is personalised — **security critical** |
| Locale / language | Text is localised |
| Currency / region | Prices or availability differ |
| Schema/format version | Always (see below) |
| Permission scope / role | The visible fields differ by role |
| Device class | Only if the payload genuinely differs; normalise to `mobile`/`desktop` |
| Feature flag variant | The response shape depends on the flag |
| Query parameters | Only the ones that affect the result — **normalise and allowlist** |

---

## The cardinality trade-off

$$\text{hit rate} \propto \frac{\text{requests}}{\text{distinct keys}}$$

Every dimension you add **multiplies** the key space and **divides** the hit rate.

| Key | Distinct keys | Hit rate |
|---|---|---|
| `product:9` | 1 | Highest |
| `product:9:en-GB` | × 20 locales | ÷ 20 |
| `product:9:en-GB:GBP` | × 5 currencies | ÷ 100 |
| `product:9:en-GB:GBP:user:44` | × 10M users | **≈ 0** |

☠️ **The over-specified key.** A key containing the user ID for content that is identical for all users gives every user their own copy — the cache holds 10 million entries where one would do, and the hit rate per entry approaches zero.

✅ **The fix: split the response.** Cache the shared part under a shared key, and the personal part under a per-user key, then compose.

```python
product   = cache.get(f"product:v3:{pid}:{locale}")      # shared, high hit rate
user_view = cache.get(f"user:{uid}:prefs")               # small, per-user
return render(product, user_view)
```

⚠️ **The under-specified key is worse.** Omitting a dimension that *does* affect the value means users get each other's data. If that dimension is the user ID, it is a **security incident**, not a bug.

---

## Naming conventions

```
<namespace>:<version>:<entity>:<id>[:<dimension>=<value>]...
```

| Rule | Reason |
|---|---|
| Use `:` as the separator | Conventional in Redis; tools display it hierarchically |
| Include an app/service namespace on a shared cluster | Prevents cross-service collisions |
| Include a **schema version** (`v3`) | Deploying a changed serialisation format is otherwise a disaster |
| Lowercase, no spaces | Avoids near-duplicate keys |
| Keep keys short | Redis stores keys in RAM; 10M × 100-byte keys = 1 GB of *keys* |
| Build keys in **one function**, never inline | The single most important practical rule |

```python
# One place, one format, always.
def product_key(pid: int, locale: str, currency: str) -> str:
    return f"prod:v3:{pid}:{locale.lower()}:{currency.upper()}"
```

☠️ **Inline key construction is the #1 source of cache bugs.** One code path writes `product:9` and another reads `products:9`; the cache silently never hits and nobody notices for months — the system just quietly costs 10× more. Centralise key building and unit-test it.

### The schema version is not optional

```
v1: {"id":9,"name":"Kettle"}
v2: {"id":9,"name":"Kettle","price_minor":2499}     ← new field
```

Deploy v2 code while v1 entries are still cached, and v2 code reads an object missing `price_minor` → `KeyError` on a fraction of requests, for the duration of the TTL.

**Bump the version in the key** and the two generations coexist harmlessly; old entries are simply never read again and get evicted. This also gives you **instant, free, total invalidation**: change `v3` to `v4` and the entire namespace is logically cleared with zero purge operations.

---

## Normalisation — the hit-rate multiplier

The same logical request can arrive in many textual forms. Normalise **before** keying.

```python
def normalize(params: dict) -> str:
    ALLOWED = {"category", "sort", "page", "limit"}      # allowlist
    DEFAULTS = {"sort": "relevance", "page": "1", "limit": "20"}
    p = {k: v for k, v in params.items() if k in ALLOWED}   # drop utm_*, fbclid...
    p = {**DEFAULTS, **p}                                   # fill defaults
    p = {k: v.strip().lower() for k, v in p.items()}        # canonical case
    return "&".join(f"{k}={p[k]}" for k in sorted(p))       # stable ordering
```

| Without normalisation | With |
|---|---|
| `?sort=price&page=1` | `page=1&sort=price` |
| `?page=1&sort=price` | `page=1&sort=price` |
| `?sort=Price&page=1&utm_source=x` | `page=1&sort=price` |
| `?sort=price` (page defaults to 1) | `page=1&sort=price` |

**Four distinct keys become one.** The **allowlist** matters most: without it, a crawler appending random parameters creates unlimited distinct keys, evicting your real hot set — a genuine denial-of-service vector.

---

## Hierarchical keys and group invalidation

You need to invalidate "everything about product 9." Options:

### ❌ `KEYS product:9:*` / `SCAN`
`KEYS` is O(N) over the entire keyspace and **blocks Redis** — never run it in production. `SCAN` is non-blocking but still O(N) and slow on large keyspaces.

### ✅ Tag sets
```
SADD  tag:product:9  "prod:v3:9:en:GBP"  "prod:v3:9:fr:EUR"  "page:product:9"
# invalidate:
SMEMBERS tag:product:9  →  DEL each  →  DEL tag:product:9
```
Explicit, exact, and O(number of tagged keys).

### ✅✅ Version prefixes (usually best)
```
GET  ver:product:9                → 12
GET  prod:v3:9:12:en:GBP          → the value

# invalidate: one atomic increment
INCR ver:product:9                → 13
# every key containing :12: is instantly orphaned and will be evicted by LRU
```

| | Tag sets | Version prefixes |
|---|---|---|
| Invalidate cost | O(k) deletes | **O(1) increment** |
| Extra read | No | Yes (one lookup — batch it or cache the version locally for 1 s) |
| Memory reclaim | Immediate | Lazy, via eviction |
| Works across layers (CDN, browser) | ❌ | ✅ — put the version in the URL |
| Race-safe | Needs care | ✅ Naturally |

🎯 **Version prefixes are the strongest general answer** and work uniformly at every cache layer, including ones you cannot purge. Say this when asked about invalidation.

---

## Security: the key IS the access control boundary

☠️ **The multi-tenant leak.** A key of `invoice:123` in a shared cache means any tenant who can request invoice 123 gets it — even if the database would have refused them. **The cache bypassed your authorization check.**

```python
# WRONG — the authorization check happened, but the cache key didn't record it
def get_invoice(invoice_id, user):
    if v := cache.get(f"invoice:{invoice_id}"):
        return v                                  # ← no auth check on the hit path!
    inv = db.get_invoice(invoice_id, owner=user.tenant_id)
    cache.set(f"invoice:{invoice_id}", inv)
    return inv

# RIGHT — the tenant is part of the key, and authorization is checked before the cache
def get_invoice(invoice_id, user):
    authorize(user, invoice_id)                   # always, before touching the cache
    key = f"invoice:v2:{user.tenant_id}:{invoice_id}"
    if v := cache.get(key):
        return v
    inv = db.get_invoice(invoice_id, owner=user.tenant_id)
    cache.set(key, inv, ex=300)
    return inv
```

**Two rules, both mandatory:**
1. **Authorize before consulting the cache**, never only on the miss path.
2. **Scope the key by tenant/user** for anything not universally public.

⚠️ Also consider **cache-key injection**: if user input flows into a key unsanitised, a user supplying `9:en:GBP` as an "ID" can craft a key that collides with another entry. Sanitise, or hash user-supplied components.

---

## Long keys and hashing

When a key must encode many dimensions (a complex search query), hash the canonical form:

```python
canonical = json.dumps(normalized_params, sort_keys=True, separators=(",", ":"))
key = f"search:v2:{hashlib.blake2b(canonical.encode(), digest_size=16).hexdigest()}"
```

⚖️ **Trade-off:** short, uniform keys and no length limits — but the key is now **opaque**. You cannot inspect the cache or invalidate by pattern. Mitigate by storing the canonical form *inside* the cached value for debugging, and by pairing with tag sets or version prefixes for invalidation.

---

## Hot keys

One key receiving a disproportionate share of traffic saturates a single shard.

| Mitigation | How |
|---|---|
| **Local cache with a 1 s TTL** | 10,000 QPS → 1 QPS per instance. ✅ Usually sufficient on its own |
| **Key splitting** | Write to `counter:9:{0..15}`, read all 16 and sum |
| **Replicate the key** | Store `hot:9:r0..r9`; readers pick one at random |
| **Serve from CDN** | If it's public content |
| **Detect dynamically** | Track top-K keys; promote them to local caches automatically |

🔢 The local-cache maths again, because it's the answer most of the time: **a 1-second TTL on a key receiving 10,000 QPS reduces load 10,000× at the cost of ≤ 1 second of staleness.**

---

## A worked key schema

```
# Entities
prod:v3:{id}:{ver}:{locale}:{currency}          TTL 300s
user:v2:{uid}                                   TTL 600s
inv:v1:{tenant}:{invoice_id}                    TTL 300s

# Versions (for group invalidation)
ver:prod:{id}                                   no TTL
ver:catalog                                     no TTL

# Computed
feed:v4:{uid}:{ver}                             TTL 60s
count:followers:{uid}                           TTL 30s
rank:leaderboard:{region}:{day}                 sorted set, TTL 86400

# Infrastructure
lock:order:{order_id}                           TTL 30s
rate:{scope}:{id}:{window}                      TTL = window
idem:{client_id}:{key}                          TTL 86400
sess:{session_id}                               TTL 1800
```

Conventions visible here: every namespace has a version; per-tenant data is tenant-scoped; every entry has a deliberate TTL; and version keys are the invalidation mechanism.

---

## ☠️ Failure modes

| Mistake | Consequence |
|---|---|
| Missing user/tenant in the key | **Cross-user data leak** |
| User ID in a key for shared data | Hit rate ≈ 0; wasted memory |
| No schema version | Deploys break on stale-format entries |
| Inline key construction | Typo → silent 0% hit rate |
| No parameter allowlist | Crawlers evict your hot set |
| `KEYS` in production | Blocks Redis; can take the whole cache down |
| Very long keys | Significant RAM overhead at scale |
| Authorization only on the miss path | The cache bypasses your access control |
| Unsanitised user input in keys | Key collision / injection |
| No group-invalidation mechanism | You cannot clear related entries; you resort to flushing everything |

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| List everything a key must encode and the test for it | ☐ |
| Explain the cardinality/hit-rate relationship with numbers | ☐ |
| Split a personalised response into shared and per-user keys | ☐ |
| Explain why schema versions in keys are mandatory | ☐ |
| Write a normalisation function with an allowlist | ☐ |
| Compare tag sets and version prefixes for group invalidation | ☐ |
| State the two mandatory security rules | ☐ |
| Explain hot keys and four mitigations | ☐ |
| Design a full key schema for a product | ☐ |

---

**← Previous** [3.3.3 CDN vs Application Cache](03-cdn-vs-app-cache.md)
**Next →** [3.3.5 Read Patterns: Fetching Data](05-read-patterns.md)
