# 4.1.2 — B-tree

> **Part 4 · Data Storage · Data Structures Behind Databases · Chapter 2 of 4**
> The structure behind every relational database you've used.

---

## 🧒 ELI5 — Explain Like I'm 5

A B-tree is a **filing cabinet with signposts**.

Imagine looking for the word "kettle" in a huge dictionary. You don't read from page 1. You open roughly in the middle, see you've landed on "M", and go back. Then you land on "H", go forward. Three or four jumps and you're there.

A B-tree does exactly that, but the signposts are written down:

- **Top signpost:** "A–M go left, N–Z go right."
- **Next signpost:** "A–F left, G–M right."
- ...and after three or four signposts, you're on the page with your word on it.

Two clever bits:

1. **Each signpost handles hundreds of choices, not two.** So even with a *billion* words, you only follow about four signposts. Four! *(That's what "high fan-out" means, and it's why B-trees are shallow.)*
2. **The final pages are chained together like a paper chain.** So once you've found "kettle," reading "kettles, kettling, kevlar..." is just following the chain — no more signpost-hopping. *(That's why range queries are fast.)*

The annoying bit: when you **add** a word, you have to squeeze it onto the right page. If the page is full, you **tear it in half** and add a new signpost above. Occasionally that signpost's page is full too, so it splits, and so on up the tree. Adding is more work than looking up.

---

## The structure

```
Level 0 (root)          [ 50 | 100 ]
                       /      |      \
Level 1        [10|25|40] [60|75|90] [110|150|200]
                /  |  \      ...          ...
Level 2      leaves: sorted (key → value or row pointer)
             ← linked → ← linked → ← linked →
```

**Invariants of a B+tree** (what real databases use):

| Rule | Meaning |
|---|---|
| All values live in the **leaves** | Internal nodes hold only keys and child pointers → higher fan-out |
| All leaves are at the **same depth** | The tree is perfectly balanced — every lookup costs the same |
| A node holds between ⌈m/2⌉ and m children | Guarantees at least 50% occupancy, so the tree can't degenerate |
| **Leaves are linked** left↔right | Range scans are sequential after the first lookup |
| One node = one **disk page** (4–16 KB) | The unit of I/O matches the unit of structure |

⚠️ "B-tree" is used loosely; databases almost always implement a **B+tree** (values in leaves, leaves linked). The distinction matters for range scans.

---

## Why it's shallow — the fan-out arithmetic

```
Page size            16 KB
Key size             16 bytes
Pointer size          8 bytes
Entries per node     16,384 / 24 ≈ 680

Depth 1:  680 keys
Depth 2:  680²   ≈ 462,000
Depth 3:  680³   ≈ 314,000,000
Depth 4:  680⁴   ≈ 214,000,000,000
```

🔢 **Four levels indexes 214 billion rows.** And because the root and the first level or two are always in the buffer pool, a point lookup on a billion-row table typically costs **one disk read**.

🎯 This is the number to quote when asked "how does a database find a row so fast?" — *"a B-tree of depth 4 covers hundreds of billions of keys, and the upper levels are cached, so it's usually one physical read."*

---

## Operations

### Lookup — O(log n)
```
find(75):
  root [50|100]      → 50 ≤ 75 < 100 → middle child
  node [60|75|90]    → found the separator, go to the leaf
  leaf               → binary search within the page → value
```
Within a page, a binary search over a sorted array — pure CPU, no I/O.

### Range scan — O(log n + k)
```
range(60, 90):
  descend to the leaf containing 60
  read entries; when the page ends, follow the sibling pointer
  repeat until past 90
```
✅ **This is the B-tree's decisive advantage over hash indexes and its edge over LSM trees.** `WHERE created_at BETWEEN ... AND ...`, `ORDER BY ... LIMIT`, and cursor pagination all rely on it.

### Insert — with splits
```
insert(85) into a full leaf [80|82|85|88]:
  1. split into [80|82] and [85|88]
  2. promote the middle key (85) to the parent
  3. if the parent is now full, split it too — recursively
  4. if the root splits, the tree grows by one level (the only way it deepens)
```

⚠️ **A split writes at least three pages** (the two halves plus the parent) — and via the WAL, considerably more. This is the source of B-tree write amplification.

### Delete — with merges
Remove the entry; if the node falls below half occupancy, borrow from a sibling or merge with it. In practice most databases **don't rebalance aggressively** — they leave partly-empty pages, which is cheaper but causes **fragmentation** over time. Hence `VACUUM` (Postgres) and `OPTIMIZE TABLE` (MySQL).

---

## Write amplification and the WAL

A single 100-byte row update causes:

```
1. WAL record appended and fsync'd                   (~100-300 bytes, sequential)
2. The 16 KB page modified in the buffer pool
3. Eventually the whole 16 KB page written to disk   ← 160x the row size
4. Possibly index page writes for every index on the table
5. MySQL only: a doublewrite buffer copy (16 KB again)
```

🔢 **Typical B-tree write amplification: 2–10×**, and much worse for small rows on large pages. This is the fundamental reason LSM trees exist.

⚠️ **Torn pages:** if the OS crashes mid-write, a 16 KB page can be partially written and become corrupt. Postgres solves this with **full-page writes** to the WAL after each checkpoint; MySQL uses the **doublewrite buffer**. Both cost extra I/O — this is why "just turn off `full_page_writes`" is dangerous advice.

---

## Clustered vs non-clustered

| | **Clustered** (InnoDB, SQL Server) | **Heap + secondary** (Postgres) |
|---|---|---|
| Table storage | Rows stored **in primary-key order inside the B-tree leaves** | Rows in an unordered heap; indexes point at (page, offset) |
| PK lookup | ✅ One traversal, row is right there | Traversal + one heap fetch |
| Secondary index lookup | Index → PK → **second traversal** of the clustered index | Index → tuple pointer → heap fetch |
| Range scan on PK | ✅ Physically sequential | Random heap access unless the table is clustered |
| PK choice matters | ✅ **Enormously** | Less so |

☠️ **The InnoDB random-primary-key trap.** With a clustered index, a **random UUID** primary key means every insert lands at a random position in the tree — random page writes, constant splits, terrible cache locality, and severe fragmentation. A **monotonic** key (auto-increment, ULID, Snowflake) makes every insert land at the right edge: sequential, minimal splits.

🔢 The measured difference on large tables is often **5–10× insert throughput**. If a design uses UUIDs as primary keys in MySQL, say this — it's a favourite follow-up. (Use UUIDv7 or ULID, which are time-ordered, if you need UUID semantics.)

⚠️ In InnoDB, secondary indexes store the **primary key** as the row pointer. A wide primary key (a 36-char UUID string) is therefore duplicated into *every* secondary index, inflating them all.

---

## Concurrency

| Mechanism | How |
|---|---|
| **Latch coupling (crabbing)** | Lock a child before releasing the parent, so a concurrent split can't corrupt the traversal |
| **B-link trees** | Each node has a right-sibling pointer, so a reader that arrives mid-split can follow it — enables lock-free reads |
| **MVCC** | Readers see a snapshot; writers create new versions. Readers never block writers |
| **Page-level latches** | Short-lived, distinct from transaction locks |

☠️ **The hot-page contention problem:** with a monotonic primary key, every insert targets the **rightmost leaf page**, which becomes a latch contention point across all writer threads. Databases mitigate this with optimised right-edge insert paths, but at extreme write rates it's a real ceiling — and it's the counter-argument to "always use a monotonic key." (The usual resolution: monotonic within a shard, sharded across many.)

---

## Fragmentation and maintenance

Over time: pages become partly empty (deletes), logically-adjacent pages become physically scattered (splits), and the index grows larger than its data warrants.

| Symptom | Fix |
|---|---|
| Index bloat | `REINDEX CONCURRENTLY` (Postgres), `OPTIMIZE TABLE` (MySQL) |
| Table bloat from dead tuples | `VACUUM` (Postgres — dead tuples are only reclaimed by vacuum) |
| Poor range-scan locality | `CLUSTER` (Postgres, takes a lock) or rebuild |
| Fill factor too high for update-heavy tables | Set `fillfactor` to 70–90% to leave room for in-page updates |

⚠️ **Postgres autovacuum is not optional.** Under heavy update/delete load, insufficient vacuuming causes unbounded table bloat and eventually **transaction ID wraparound**, which forces a shutdown. It's one of the most common Postgres production incidents.

---

## When B-trees are the right choice

| ✅ Use a B-tree engine | ❌ Prefer LSM |
|---|---|
| Read-heavy or balanced workloads | Write-heavy (10:1 writes or more) |
| Range scans, `ORDER BY`, cursor pagination | Mostly point lookups |
| Transactions and strong consistency | Eventual consistency is acceptable |
| Predictable, low read latency | Write throughput matters more than read latency |
| Data fits comfortably with the working set cached | Very large datasets with a small hot set |
| Frequent in-place updates of existing rows | Append-mostly, time-ordered data |

🎯 **The one-line justification:** *"Reads dominate and we need range queries and transactions, so a B-tree engine — Postgres. The write amplification is acceptable at our write rate."*

---

## Practical tuning

| Setting | Guidance |
|---|---|
| `shared_buffers` / `innodb_buffer_pool_size` | 25–40% (PG) or 50–70% (MySQL) of RAM. **This is the single most impactful setting** |
| Page size | Usually leave at default (8 KB PG, 16 KB InnoDB) |
| `fillfactor` | 70–90% for update-heavy tables; 100% for append-only |
| `checkpoint_timeout` / `max_wal_size` | Longer = fewer full-page writes but longer recovery |
| Index count | Every extra index is a write cost — audit with `pg_stat_user_indexes` |
| Composite index column order | Most selective *or* leftmost-matching first, based on actual query patterns |

**The buffer pool is the whole game:** if the working set fits in it, lookups are memory-speed and the B-tree's write cost is the only downside. If it doesn't, every lookup is a disk read and performance falls off a cliff. Sizing the buffer pool against the working set is the highest-leverage database tuning there is.

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| Draw a B+tree and state its four invariants | ☐ |
| Do the fan-out arithmetic and state the depth for a billion rows | ☐ |
| Explain range scans via linked leaves | ☐ |
| Explain splits and where write amplification comes from | ☐ |
| Explain torn pages and full-page writes | ☐ |
| Compare clustered and heap storage | ☐ |
| Explain the random-UUID primary key problem and its fix | ☐ |
| Explain right-edge latch contention | ☐ |
| Explain vacuum, bloat, and why autovacuum matters | ☐ |
| State when to choose B-tree over LSM | ☐ |

---

**← Previous** [4.1.1 Data Structures Behind Databases](01-data-structures-behind-databases.md)
**Next →** [4.1.3 SSTable](03-sstable.md)
