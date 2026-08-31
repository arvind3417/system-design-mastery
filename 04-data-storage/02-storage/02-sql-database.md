# 4.2.2 — SQL Database

> **Part 4 · Data Storage · Storage · Chapter 2 of 9**
> Relational databases: what they guarantee, what they cost, and how far they actually scale.

---

## 🧒 ELI5 — Explain Like I'm 5

A relational database is **a very strict, very clever filing system**.

**Strict**, because it enforces rules you write down once and it never lets anyone break them:

- "Every order must belong to a real customer." Try to file an order for customer #999 who doesn't exist, and it **refuses**. *(Foreign keys.)*
- "No two people can have the same email." It **refuses** the second one. *(Unique constraints.)*
- "Either the money leaves Ann's account AND arrives in Bob's, or neither happens." If the lights go out halfway through, it **undoes** the half that happened. *(Transactions.)*

**Clever**, because you don't tell it *how* to find things. You say **what you want** — "all orders over £100 from customers in London, newest first" — and it works out the fastest way to get it, using whatever indexes exist. *(The query planner.)*

The catch: all that checking and cleverness happens on **one machine** for writes. You can make that machine bigger, and you can add copies for reading, but there is ultimately one place where the strictness is enforced. Getting past that is where all the difficulty lives.

---

## What you're actually buying

| Property | What it means in practice |
|---|---|
| **ACID transactions** | Multi-row changes are all-or-nothing, even across tables |
| **Declarative queries** | Say what, not how; the planner picks the strategy |
| **Joins** | Combine normalised data at read time — no denormalisation needed |
| **Constraints** | The database refuses invalid data, whatever the application does |
| **Schema** | Structure is enforced and self-documenting |
| **Maturity** | 40+ years of tooling, expertise, backups, monitoring, and known failure modes |

🎯 **The under-appreciated one is constraints.** A `CHECK`, `NOT NULL`, `UNIQUE`, or foreign key is enforced no matter which service, script, migration, or intern writes the data. Application-level validation is bypassed the moment anything writes outside the application — which always eventually happens.

---

## ACID, precisely

| | Meaning | How it's implemented |
|---|---|---|
| **Atomicity** | All or nothing | WAL + rollback |
| **Consistency** | Constraints always hold | Checks at commit |
| **Isolation** | Concurrent transactions don't corrupt each other | Locking or MVCC |
| **Durability** | Committed means committed | WAL `fsync` |

⚠️ **"Consistency" in ACID is not the same as "consistency" in CAP.** ACID-C means *your declared constraints hold*. CAP-C means *all replicas agree*. Interviewers sometimes probe this; conflating them is a common error.

### Isolation levels — know the anomalies

| Level | Dirty read | Non-repeatable read | Phantom read | Write skew |
|---|---|---|---|---|
| Read Uncommitted | ✅ possible | ✅ | ✅ | ✅ |
| **Read Committed** (PG/Oracle default) | ❌ | ✅ | ✅ | ✅ |
| **Repeatable Read** (MySQL default) | ❌ | ❌ | ✅* | ✅ |
| **Serializable** | ❌ | ❌ | ❌ | ❌ |

\* Postgres's Repeatable Read (snapshot isolation) prevents phantoms; the SQL standard doesn't require it.

☠️ **Write skew is the one that bites in production**, and it's invisible at anything below Serializable:

```sql
-- Rule: at least one doctor must remain on call.
-- Two doctors, both on call, both try to go off call simultaneously.
-- Transaction A:  SELECT count(*) FROM doctors WHERE on_call = true;  -- sees 2, OK
-- Transaction B:  SELECT count(*) FROM doctors WHERE on_call = true;  -- sees 2, OK
-- A: UPDATE doctors SET on_call = false WHERE id = 1;   ✓
-- B: UPDATE doctors SET on_call = false WHERE id = 2;   ✓
-- Result: ZERO doctors on call. Both transactions were "correct".
```

**Fixes:** `SERIALIZABLE` isolation, `SELECT ... FOR UPDATE` to lock the rows read, or a materialised constraint (a counter row) that forces a conflict.

🎯 Being able to describe write skew concretely is a strong senior signal — most candidates only know dirty reads.

### MVCC

Postgres and MySQL (InnoDB) use **multi-version concurrency control**: writers create new row versions; readers see a consistent snapshot. **Readers never block writers, and writers never block readers.**

⚠️ The cost is **dead tuples**: old versions accumulate until vacuumed. Postgres's autovacuum must keep up, or you get table bloat and, eventually, transaction-ID wraparound. This is the single most common Postgres operational problem.

---

## Schema design essentials

### Normalisation, then selective denormalisation

Normalise to 3NF by default: each fact lives in exactly one place, so updates can't create contradictions.

**Denormalise deliberately, for a measured read pattern:**

| Denormalisation | When |
|---|---|
| Duplicate a column to avoid a join | The join is on the hot path and measured slow |
| Counter columns (`comment_count`) | Counting on read is too expensive |
| Materialised views | Complex aggregations read often |
| JSONB for sparse/variable attributes | Genuinely schema-less parts of the model |

⚠️ Every denormalisation creates an update obligation. If `comment_count` is maintained by application code, it will drift. Maintain it with a trigger or a scheduled reconciliation — and have a job that detects drift.

### Choosing keys

| Key type | Pros | Cons |
|---|---|---|
| `BIGSERIAL` / auto-increment | Compact, monotonic (great for clustered indexes) | Leaks volume; collides across shards |
| **UUIDv4** | Globally unique, generated anywhere | ❌ Random → terrible for clustered indexes; 16 bytes |
| **UUIDv7 / ULID** | ✅ Globally unique **and** time-ordered | Slightly larger than an int |
| Snowflake | Time-ordered, sortable, compact | Needs a generator service or worker IDs |
| Natural key (email) | No extra column | Changes; wide; duplicated into every index |

🎯 **The recommendation: UUIDv7 or ULID.** You get global uniqueness (essential for sharding and for offline-generated IDs) without the random-insert penalty of UUIDv4 ([B-tree](../01-data-structures-behind-databases/02-b-tree.md#clustered-vs-non-clustered)).

### Data types that matter

| Use | Not |
|---|---|
| `BIGINT` for money in minor units | `FLOAT`/`DOUBLE` — never |
| `NUMERIC`/`DECIMAL` if you need fractional precision | `FLOAT` |
| `TIMESTAMPTZ` (UTC) | `TIMESTAMP` without a zone |
| `TEXT` with a `CHECK` on length | Arbitrary `VARCHAR(255)` |
| Native `ENUM` or a lookup table + FK | Magic integers |
| `JSONB` with GIN indexes for sparse attributes | JSON as a way to avoid designing a schema |

---

## Query performance

### The planner
The database chooses an execution plan from statistics. Your job is to give it good indexes and accurate statistics.

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```

| Plan node | Meaning |
|---|---|
| `Seq Scan` | Reads the whole table. Fine on small tables, a disaster on large ones |
| `Index Scan` | Uses an index, then fetches rows |
| `Index Only Scan` | ✅ The index covers the query — no table access at all |
| `Bitmap Heap Scan` | Many matching rows; batches the heap fetches |
| `Nested Loop` | Good when the outer side is small |
| `Hash Join` | Good for large unsorted joins |
| `Merge Join` | Good when both sides are already sorted |

⚠️ **Look at the ratio of estimated to actual rows.** A plan expecting 10 rows and getting 1,000,000 means the statistics are stale (`ANALYZE`) or the predicate is unindexable — and the planner has chosen a strategy that's catastrophically wrong.

### The high-value fixes

| Problem | Fix |
|---|---|
| N+1 queries | Batch with `WHERE id = ANY($1)`, or a join |
| Missing index on a filter/join column | Add it — but audit for unused ones |
| `OFFSET` pagination on a large table | Cursor pagination ([Pagination](../../01-introduction/02-api-design/03-pagination.md)) |
| `SELECT *` fetching unused columns | Select what you need; enables index-only scans |
| Function on an indexed column (`WHERE lower(email)=`) | An expression index on `lower(email)` |
| `LIKE '%term%'` | Trigram index (`pg_trgm`), or a search engine |
| Long transactions | They hold locks and block vacuum — keep them short |
| Lock contention on a hot row | Batch updates, or a sharded counter |

🎯 **The single most common real-world fix is the N+1 query**, and it's worth naming unprompted: 200 sequential queries at 1 ms each is 200 ms of pure latency that one batched query eliminates.

---

## How far does one node scale?

🔢 Honest figures for a well-tuned modern server (32+ cores, 128+ GB RAM, NVMe):

| Metric | Realistic ceiling |
|---|---|
| Simple indexed reads | 20,000–50,000 QPS |
| Simple writes (with `fsync`) | 5,000–20,000 QPS |
| Complex joins | 100–2,000 QPS |
| Dataset size | 5–50 TB (working set must fit in RAM for good performance) |
| Concurrent connections | **Hundreds**, not thousands — use a pooler |

⚖️ **This is much further than people assume.** Most systems never exceed it. Stack Overflow, Shopify's early years, and countless large SaaS products ran on a single primary for a long time. **Exhaust vertical scaling and caching before sharding.**

### The scaling ladder

```
1. Index and query tuning         ← usually 10-100x, free
2. Cache                          ← removes 90%+ of reads
3. Bigger machine                 ← 2-10x, no code change
4. Read replicas                  ← scales reads only
5. Connection pooling             ← removes the connection ceiling
6. Partitioning (within one node) ← smaller indexes, easier maintenance
7. Sharding                       ← scales writes; the big complexity jump
8. A distributed SQL database     ← Spanner, CockroachDB, Vitess, Citus
```

⚠️ **Steps 1 and 2 are usually worth more than steps 3–7 combined**, and cost the least. Reaching for sharding before profiling queries is the classic mistake.

### Connection pooling is not optional

☠️ Postgres allocates a **process per connection** (~5–10 MB each). 10 app servers × 100-connection pools = 1,000 connections = failure. The database sits at 5% CPU while everything times out.

**Fix:** PgBouncer in transaction mode, RDS Proxy, or ProxySQL. Real pool size ≈ `cores × 2 + effective_spindles` — usually **tens**. More connections past that point make throughput *worse*, not better.

---

## Partitioning (within one database)

```sql
CREATE TABLE events (id BIGINT, created_at TIMESTAMPTZ, ...)
  PARTITION BY RANGE (created_at);
CREATE TABLE events_2026_08 PARTITION OF events
  FOR VALUES FROM ('2026-08-01') TO ('2026-09-01');
```

| Benefit | Detail |
|---|---|
| **Partition pruning** | Queries with a date filter scan only relevant partitions |
| **Cheap deletion** | `DROP TABLE events_2025_01` is instant; `DELETE` of a billion rows is not |
| Smaller indexes | Per-partition indexes fit in memory |
| Parallel maintenance | Vacuum/analyze per partition |

⚠️ Partitioning does **not** scale writes beyond one node — it's an organisational and maintenance win, not a distribution one. Confusing partitioning with sharding is a common error.

---

## Migrations without downtime

The **expand/contract** pattern:

```
1. Add the new column, nullable, with a default that doesn't rewrite the table
2. Deploy code that writes BOTH old and new
3. Backfill in batches (never one giant UPDATE — it holds locks and bloats the table)
4. Deploy code that reads the new column
5. Deploy code that stops writing the old column
6. Drop the old column
```

Each step is independently deployable and reversible.

☠️ **Migrations that lock a large table are a leading cause of self-inflicted outages.** In Postgres, use `CREATE INDEX CONCURRENTLY`, add constraints as `NOT VALID` then `VALIDATE`, and **always** set a short `lock_timeout` so a migration fails fast rather than queueing behind a long transaction and blocking every subsequent query.

---

## When SQL is the wrong tool

| Situation | Better |
|---|---|
| Write rate exceeds one node and the model is key-ordered | Cassandra, DynamoDB |
| Petabyte analytics | ClickHouse, BigQuery |
| Full-text search with ranking and facets | Elasticsearch |
| Large binary objects | Object storage |
| Sub-millisecond point reads | Redis |
| Deep graph traversals | Neo4j |
| Truly schema-less, wildly varying documents | Document store (but consider JSONB first) |

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| List what relational databases actually give you | ☐ |
| Distinguish ACID-C from CAP-C | ☐ |
| Name the four isolation levels and their anomalies | ☐ |
| Describe write skew with a concrete example and three fixes | ☐ |
| Explain MVCC and why vacuum matters | ☐ |
| Choose a primary key type and justify it | ☐ |
| Read an `EXPLAIN ANALYZE` plan and spot a bad estimate | ☐ |
| Quote realistic single-node QPS and size ceilings | ☐ |
| Explain the connection-pooling ceiling | ☐ |
| Distinguish partitioning from sharding | ☐ |
| Describe an expand/contract migration | ☐ |

---

**← Previous** [4.2.1 Introduction to Storage](01-introduction-to-storage.md)
**Next →** [4.2.3 Introduction to NoSQL Databases](03-nosql-database.md)
