# 4.2.4 — Key-value Database

> **Part 4 · Data Storage · Storage · Chapter 4 of 9**
> The simplest data model, and therefore the fastest and most scalable.

---

## 🧒 ELI5 — Explain Like I'm 5

A key-value store is **a cloakroom**.

You hand over your coat, they give you a **ticket with a number**. Later you show the ticket and get your coat back. That's it. That's the whole system.

Because it's so simple, it's incredibly fast — no searching, no thinking, just "ticket 47 → shelf 47."

And because it's so simple, it **scales beautifully**. Ten cloakrooms? Easy: tickets 1–100 go to cloakroom 1, 101–200 to cloakroom 2. Nobody needs to coordinate; each cloakroom minds its own shelves.

But notice what you **cannot** ask a cloakroom:

- *"Which coats are blue?"* — they'd have to open every single one.
- *"Whose coat is this?"* — tickets only go one way.
- *"Give me all coats handed in after 3pm."* — no such list exists.

**You can only ask by ticket number.** If you'll ever want "all the blue coats," you must decide that *in advance* and keep a separate list of blue-coat tickets.

That's the deal: **the fastest possible lookup, in exchange for only ever being able to look up one way.**

---

## The model

```
key (string/bytes)  →  value (opaque bytes)

user:44             →  {"name": "Ann", "email": "..."}
session:a7f3c1      →  {"user_id": 44, "expires": ...}
counter:views:99    →  1204
```

| Operation | Complexity |
|---|---|
| `GET key` | O(1) |
| `PUT key value` | O(1) |
| `DELETE key` | O(1) |
| `SCAN` / range | ❌ Not supported (or O(n)) |
| Query by value | ❌ Not supported |

**The database does not interpret the value.** That's what makes it fast — no parsing, no indexing, no schema validation — and what makes it limited.

---

## Why it scales so well

Because there are **no relationships between keys**, any key can live on any node with no coordination:

```
node = consistent_hash(key)
```

- No joins → no cross-node queries.
- No transactions across keys → no distributed locking.
- No secondary indexes (by default) → no index maintenance on write.
- Adding a node moves ~1/N of keys and nothing else changes.

🔢 **This is genuinely linear scaling.** Doubling the nodes doubles the capacity, which is not true of relational databases and only approximately true of document stores.

---

## The two archetypes

### Redis — in-memory, rich data structures

| | |
|---|---|
| **Latency** | 0.1–1 ms; 100k+ ops/s per node |
| **Data structures** | Strings, hashes, lists, sets, sorted sets, streams, bitmaps, HyperLogLog, geospatial |
| **Persistence** | RDB snapshots + AOF (`everysec` by default — up to 1 s loss) |
| **Threading** | Single-threaded for commands (I/O threads in 6+) |
| **Scaling** | Redis Cluster (16,384 slots) or client-side sharding |
| **Best at** | Caching, sessions, counters, rate limiting, leaderboards, queues, locks, pub/sub |

🎯 **Redis's real differentiator is not speed — it's the data structures.** Atomic `INCR`, sorted sets for leaderboards, `SETNX` for locks, HyperLogLog for approximate distinct counts, and streams for logs. Those turn many multi-step application problems into single atomic operations.

```
INCR    counter:views:99                        # atomic counter
ZADD    leaderboard 1500 "player:44"            # sorted set
ZREVRANGE leaderboard 0 9 WITHSCORES            # top 10, O(log n)
SET     lock:order:88 <token> NX EX 30          # distributed lock with expiry
PFADD   uniques:2026-08-31 "user:44"            # HyperLogLog, ~12KB for millions
XADD    events * type click user 44             # stream
```

⚠️ **Redis is not durable by default.** `appendfsync everysec` can lose a second of writes; replication is asynchronous, so a failover can lose more. Fine for a cache; **not fine** for anything you can't recompute. Use MemoryDB (durable multi-AZ log) or a real database if durability matters.

### DynamoDB — managed, disk-backed, unbounded

| | |
|---|---|
| **Latency** | Single-digit ms (p99 ~10 ms) |
| **Scale** | Effectively unlimited; auto-partitions |
| **Model** | Partition key (+ optional sort key) — so it's really wide-column-ish |
| **Consistency** | Eventual by default; strongly consistent reads on request (2× cost) |
| **Transactions** | Up to 100 items, ACID, 2× cost |
| **Indexes** | LSI (same partition key) and GSI (different key, eventually consistent) |
| **Pricing** | Per read/write capacity unit, or on-demand |

**DynamoDB's composite key is the important detail:**
```
Partition key: user_id      → determines the physical partition
Sort key:      created_at   → items within a partition are sorted, enabling range queries
Query:  user_id = 44 AND created_at > '2026-08-01'   ✅ efficient
Query:  total > 100                                   ❌ full scan
```

☠️ **The hot partition problem is DynamoDB's characteristic failure.** Capacity is allocated per partition. If one partition key takes most of the traffic, that partition throttles while the table has spare capacity overall. Fixes: add a random suffix to spread writes (`user:44:0..9`), choose a higher-cardinality key, or use adaptive capacity (which helps but doesn't eliminate it).

---

## What key-value stores are for

| Use case | Why KV fits | Notes |
|---|---|---|
| **Cache** | O(1), TTL, eviction policies | The canonical use |
| **Sessions** | Key = session ID; TTL built in | Enables stateless app servers |
| **Rate limiting** | Atomic `INCR` + `EXPIRE` | See [Rate Limiting Patterns](../../07-patterns-and-templates/01-patterns/07-rate-limiting-patterns.md) |
| **Counters** | Atomic increments without read-modify-write | Views, likes |
| **Leaderboards** | Sorted sets: O(log n) rank queries | Redis-specific |
| **Distributed locks** | `SET NX EX` | ⚠️ Needs fencing tokens for correctness |
| **Feature flags / config** | Small, hot, read constantly | Local cache in front |
| **Idempotency keys** | `key → stored response`, with TTL | Payments, retries |
| **Deduplication** | Sets, or Bloom filters | Event processing |
| **Job queues** | Lists or streams | [Redis Queue Tutorial](../../02-microservices-and-dataflow/02-asynchronous-communication/04-redis-queue-tutorial.md) |
| **Shopping carts** | One key per user | Fits perfectly |
| **User profiles at scale** | Key = user ID | DynamoDB-style |

---

## Working around the limitations

### "I need to query by a second field"

| Approach | How | Cost |
|---|---|---|
| **Secondary index key** | Store `email:ann@x.com → user:44` as a second key | You maintain it; two writes; no atomicity between them |
| **Native GSI** | DynamoDB global secondary index | Managed, but eventually consistent and costs extra capacity |
| **Search engine** | Project into Elasticsearch via CDC | Full query flexibility; another system |
| **Scan** | ❌ | O(n) — acceptable only for tiny datasets or offline jobs |

### "I need range queries"
Use a **sort key** (DynamoDB) or a **sorted set** (Redis `ZRANGEBYSCORE`). If neither fits, a key-value store is the wrong choice — you want a wide-column store.

### "I need transactions across keys"
DynamoDB `TransactWriteItems`, Redis `MULTI`/Lua (same-slot only in Cluster), or restructure so the atomic unit is a single key. **The last option is usually best** — design the aggregate so one key holds everything that must change together.

### "My values are getting large"
Keep values small (< 100 KB, ideally < 10 KB). Large values block Redis and inflate DynamoDB costs (which are charged per 4 KB read / 1 KB write unit). Use the **claim check** pattern: store the payload in object storage, keep the reference in the KV store.

---

## Key design

Same discipline as [cache key design](../../03-scaling-services/03-caching/04-cache-key-design.md):

```
<namespace>:<version>:<entity>:<id>[:<dimension>]

user:v2:44
session:a7f3c1
cart:v1:44
rate:api:user:44:2026-08-31T10
lock:order:88
idem:client7:8f1c-a9
```

| Rule | Reason |
|---|---|
| Include a version | Serialisation changes don't break on old data |
| Include the tenant for multi-tenant data | **Security** — key scoping is access control |
| Keep keys short | Keys live in RAM; 10M × 100 B = 1 GB of keys alone |
| Build keys in one function | Typos cause silent 0% hit rates |
| Use `{hash tags}` for co-located keys in Redis Cluster | Multi-key operations need the same slot |

---

## Operational notes

| Concern | Redis | DynamoDB |
|---|---|---|
| Capacity planning | Size RAM for the working set + 30% | Provisioned RCU/WCU, or on-demand |
| Eviction | `maxmemory-policy allkeys-lru` for caches | Not applicable (disk-backed) |
| Hot keys | Local cache, key replication | Write sharding with suffixes |
| Big keys | `redis-cli --bigkeys`; blocks the server | Item size limit 400 KB |
| Backups | RDB/AOF | Point-in-time recovery, on-demand backups |
| Multi-region | Active-active (Redis Enterprise) or independent per region | Global tables (multi-master, last-writer-wins) |
| Cost driver | RAM (~$3–5/GB-month) | Request units and storage |

⚠️ **DynamoDB cost modelling matters.** Reads are billed per 4 KB and writes per 1 KB; a strongly-consistent read costs 2×; a transaction costs 2×. A design that reads a 100 KB item on every request costs 25 read units per request — at 10,000 QPS that's a serious bill. **Design item size deliberately.**

---

## When NOT to use a key-value store

| Situation | Better |
|---|---|
| Queries by many different fields | Relational |
| Ad-hoc analytics | Columnar |
| Relationships and joins | Relational or graph |
| Full-text search | Search engine |
| Large blobs | Object storage |
| Complex multi-key transactions | Relational |
| You don't yet know the access patterns | Relational — it lets you change your mind |

🎯 **That last row is the practical one.** A KV store forces you to commit to your access patterns on day one. If the product is still evolving, that commitment is expensive.

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| State the model and its three supported operations | ☐ |
| Explain precisely why KV stores scale linearly | ☐ |
| Name six Redis data structures and a use for each | ☐ |
| State Redis's actual durability guarantees | ☐ |
| Explain DynamoDB's partition + sort key model | ☐ |
| Explain the hot partition problem and its fixes | ☐ |
| Give four ways to query by a secondary field | ☐ |
| Explain the claim-check pattern for large values | ☐ |
| Compute DynamoDB cost from item size and QPS | ☐ |
| Name seven situations where KV is the wrong choice | ☐ |

---

**← Previous** [4.2.3 Introduction to NoSQL](03-nosql-database.md)
**Next →** [4.2.5 Document Database](05-document-database.md)
