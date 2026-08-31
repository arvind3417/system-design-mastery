# 5.2.1 — Database Partitioning

> **Part 5 · Scaling Data Storage · Data Partitioning · Chapter 1 of 4**
> Splitting data across machines. The only way to scale writes — and the biggest complexity jump you can make.

---

## 🧒 ELI5 — Explain Like I'm 5

One library got too big. So you build **ten libraries** and split the books between them.

Now every question has a new first step: **"which library?"**

- "Give me book #4,285" → easy, if you decided in advance that book numbers 4,000–4,999 live in library 4. **One trip.**
- "How many books are there in total?" → you must ask **all ten** libraries and add up the answers. Ten trips for one number.
- "Move this book from library 3 to library 7 *and* update the catalogue, all as one indivisible action" → **very hard.** Two buildings, two librarians, no shared clipboard.

And the mistake everyone makes: **splitting them badly.** If you split by *first letter of the author's surname*, library "S" gets Smith, Singh, Sharma, Scott — it's enormous and always busy, while library "Q" has four books and a bored librarian. **One library is on fire; nine are idle.**

**How you split is everything.** Splitting is easy. Splitting *well*, and being able to change your mind later, is the hard part.

---

## Sharding vs partitioning

| Term | Meaning |
|---|---|
| **Partitioning** | Splitting a table into pieces — often **within one database node** |
| **Sharding** | Splitting data across **separate database nodes** |
| **Vertical partitioning** | Splitting *columns* into separate tables (rarely-used or large columns moved out) |
| **Horizontal partitioning / sharding** | Splitting *rows* |

⚠️ These are used loosely. **Partitioning within a node scales maintenance and query pruning; sharding across nodes scales writes and storage.** Confusing them is a common error — say which you mean.

---

## When sharding is genuinely necessary

```
□ Write throughput exceeds one node (≈10-20k sustained writes/s) AFTER tuning
□ Dataset exceeds what one machine can store and index (≈10-50 TB)
□ The working set can't fit in RAM even after archiving
□ Regulatory data residency requires per-region storage
```

☠️ **If none of these is true, don't shard.** Go back up the [scaling ladder](../01-data-replication/01-database-scaling.md): query tuning, caching, vertical scaling, replicas, pooling, partitioning, archiving, functional split. Those give more, cost less, and are reversible.

---

## Partitioning strategies

### 1. Range partitioning

```
shard 1: user_id 1        – 1,000,000
shard 2: user_id 1,000,001 – 2,000,000
shard 3: created_at 2026-01 to 2026-06
```

| ✅ | ❌ |
|---|---|
| Range queries hit one shard | ☠️ **Hotspots**: with time or sequential IDs, the newest shard takes *all* writes |
| Easy to reason about and to add shards | Uneven distribution as data grows |
| Rebalancing by splitting a range is natural | |

**Good for:** time-series where you also delete by time (drop the whole old partition). **Bad for:** anything with a monotonically increasing key, unless combined with another dimension.

### 2. Hash partitioning

```
shard = hash(user_id) % N
```

| ✅ | ❌ |
|---|---|
| ✅ Even distribution | Range queries hit **every** shard |
| No hotspots from sequential keys | ☠️ Changing N remaps ~everything (use consistent hashing) |
| Simple | Loses locality entirely |

**The default choice** for entity-keyed data.

### 3. Consistent hashing

Hash both keys and nodes onto a ring; a key belongs to the next node clockwise. **Adding or removing a node moves only ~1/N of keys**, instead of ~all of them. Requires **virtual nodes** for even distribution. Full treatment: [Consistent Hashing](03-consistent-hashing.md).

### 4. Directory / lookup-based

```
lookup service:  user_id → shard_id
```

| ✅ | ❌ |
|---|---|
| ✅ Total flexibility — move any key anywhere | The lookup service becomes critical infrastructure |
| Easy rebalancing, per-key placement | Extra hop (cache it aggressively) |
| Supports heterogeneous shards and per-tenant placement | Another thing to make highly available |

**Used by:** Vitess (VSchema), many large multi-tenant SaaS systems, Pinterest, Notion. **Often the right answer for multi-tenant** because you can move a huge customer to their own shard without changing anything else.

### 5. Geographic

```
EU users → EU shard;  US users → US shard
```
Driven by latency and legal residency rather than by capacity. Cross-region queries become expensive; some data may legally not be movable.

---

## Choosing the shard key — the highest-stakes decision

A good key has **four** properties:

| Property | Why | Failure if missing |
|---|---|---|
| **High cardinality** | Enough distinct values to spread across shards | Few shards used |
| **Even distribution** | No value dominates | Hot shard |
| **Present in most queries** | Queries can be routed to one shard | Every query is a scatter-gather |
| **Stable** | The value never changes | Moving a row between shards is painful |

| Candidate key | Verdict |
|---|---|
| `user_id` (hashed) | ✅ Usually the best for consumer apps — high cardinality, even, present in most queries |
| `tenant_id` | ✅ Natural for B2B, ⚠️ but tenant sizes vary enormously (one customer = one shard's worth) |
| `order_id` | ✅ Even; ❌ but "all orders for a user" becomes scatter-gather |
| `created_at` | ❌ All writes land on the newest shard |
| `country` | ❌ Low cardinality and wildly skewed |
| `email` | ✅ Even, but it changes — violates stability |
| Composite `(tenant_id, user_id)` | ✅ Locality within a tenant, spread across tenants |

🎯 **The decisive question: "what does the most common query filter on?"** If 90% of queries include `user_id`, shard on `user_id` and 90% of queries touch one shard. Shard on anything else and 90% of queries touch every shard — which is *worse* than not sharding.

☠️ **The B2B tenant trap:** sharding by `tenant_id` looks obvious until one customer is 40% of your data. Solutions: give large tenants their own shard via a directory, or use a composite key that splits a large tenant internally.

---

## Routing

| Approach | Notes |
|---|---|
| **Application-level** | The app computes the shard. Simple, but every service must implement it identically |
| **Proxy** (Vitess, Citus, ProxySQL) | ✅ Transparent to the app; handles routing, scatter-gather, and rebalancing |
| **Client library** | Shared logic; version skew is a risk |
| **Native** (MongoDB `mongos`, Cassandra, DynamoDB) | ✅ Built into the database |

```python
def shard_for(user_id):
    return SHARDS[consistent_hash(user_id) % len(SHARDS)]

def get_user(user_id):
    return shard_for(user_id).query("SELECT * FROM users WHERE id=%s", user_id)

def get_all_users_count():                     # ☠️ scatter-gather
    return sum(s.query("SELECT count(*) FROM users") for s in SHARDS)
```

🎯 **Targeted vs broadcast is the operational fact that matters.** A query including the shard key touches one node (fast, scalable). A query without it touches all N (latency = the slowest shard, and load = N×). Count how many of your queries are which.

---

## What you lose

### Cross-shard joins
Gone. Options: denormalise so the join isn't needed; do an application-side join (fetch from both, merge); or **colocate** related data by using the same shard key (Citus's `distribution_column`).

### Cross-shard transactions
Gone. Options: redesign so the atomic unit is within one shard (**by far the best**); use a saga with compensations; or two-phase commit (blocking, fragile, avoid).

🎯 **"Design so that transactions never cross shards"** is the correct answer, and it's a *data modelling* answer, not an infrastructure one. Shard by `user_id` and a user's orders, addresses, and payments all live together — so a transaction touching all three stays local.

### Global uniqueness
Auto-increment breaks. Use Snowflake, ULID/UUIDv7, or per-shard ranges. ([Unique ID Generators](../../07-patterns-and-templates/01-patterns/06-unique-id-generators.md))

### Global aggregates and sorting
`SELECT count(*)`, `ORDER BY ... LIMIT`, and pagination across shards all become scatter-gather-and-merge. For a global top-N you must fetch N from **every** shard and merge — reading `N × shards` rows to return `N`.

### Rebalancing
Adding a shard means moving data while serving traffic. **Plan for it from day one** (see below) or it becomes a multi-month project.

### Operational multiplication
N backups (not mutually consistent), N sets of migrations, N monitoring targets, N failover configurations.

---

## Rebalancing: plan it before you need it

☠️ **The naive approach — `shard = hash(key) % N`** — remaps ~90% of keys when N changes from 10 to 11. Every cache is cold, every read is a miss, and you must physically move nearly all the data.

| Strategy | How | Rebalance cost |
|---|---|---|
| **Consistent hashing** | Ring with virtual nodes | ~1/N of keys move |
| **Logical shards** (✅ best) | Create 1,024 logical shards on day one; map many-to-one onto physical nodes | Move whole logical shards — no rehashing ever |
| **Directory** | Per-key or per-range placement | Move anything, anywhere |
| **Range split** | Split a hot range into two | Localised |

🎯 **The logical-shards technique is the one to name.** Create far more logical shards than you'll ever have machines (1,024 or 4,096), map them onto physical nodes, and rebalancing becomes "move logical shard 317 from node 2 to node 5" — a data copy, with no key remapping. **Vitess, Citus, and DynamoDB all work this way**, and adopting it on day one costs nothing while retrofitting it costs a migration.

### The live-migration procedure

```
1. Mark shard S as "migrating"; record the current position
2. Copy the data to the new node (bulk)
3. Stream ongoing changes (CDC) until caught up
4. Briefly pause writes to S (or use a double-write window)
5. Switch the routing entry
6. Resume; verify; delete the source data after a safety period
```

---

## Multi-tenant sharding

| Model | Isolation | Cost | Fit |
|---|---|---|---|
| Shared tables + `tenant_id` | Weakest | ✅ Cheapest | Many small tenants |
| Schema per tenant | Medium | Medium | Hundreds of tenants; migrations × N become painful |
| Database per tenant | ✅ Strongest | Highest | Enterprise customers; regulated data |
| **Hybrid** | ✅ Practical | Balanced | Small tenants share; large tenants get dedicated shards |

🎯 **The hybrid is what mature SaaS platforms do**, and it requires a directory-based router — which is another argument for choosing directory routing early.

⚠️ For shared tables, enforce isolation in the **database** (Postgres row-level security), not only in application code. One forgotten `WHERE tenant_id = ?` is a cross-tenant data breach.

---

## Alternatives that avoid sharding

| Alternative | When |
|---|---|
| **Vertical scaling** | Almost always try first |
| **Archiving cold data** | Often shrinks the working set enough to remove the pressure |
| **Functional split** | Different tables to different databases — no key changes needed |
| **Read replicas + cache** | If reads are the pressure |
| **Distributed SQL** (CockroachDB, Spanner, Vitess, Citus) | ✅ Gives sharding *with* SQL semantics and automatic rebalancing |
| **A managed sharded store** (DynamoDB, Cassandra) | Sharding is built in and invisible |

⚖️ **Manual application-level sharding of Postgres is the option to reach for last**, not first. Citus, Vitess, or a natively distributed database give you the same scaling with far less bespoke machinery — and interviewers respect knowing that.

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| Distinguish partitioning from sharding | ☐ |
| List the four conditions that justify sharding | ☐ |
| Compare five partitioning strategies | ☐ |
| State the four properties of a good shard key | ☐ |
| Evaluate seven candidate keys | ☐ |
| Explain targeted vs broadcast queries | ☐ |
| Explain the B2B tenant-size trap | ☐ |
| List six things you lose | ☐ |
| Explain "design so transactions never cross shards" | ☐ |
| Describe logical shards and why they're the day-one choice | ☐ |
| Give the live-migration procedure | ☐ |
| Name six alternatives to sharding | ☐ |

---

**← Previous** [5.1.5 Change Data Capture](../01-data-replication/05-change-data-capture.md)
**Next →** [5.2.2 Advanced Database Partitioning Techniques and Key Selection](02-advanced-partitioning.md)
