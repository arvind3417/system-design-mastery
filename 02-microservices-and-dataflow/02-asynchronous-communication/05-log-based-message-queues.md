# 2.2.5 — Log-based Message Queues

> **Part 2 · Microservices & Data Flow · Asynchronous Communication · Chapter 5 of 7**
> The idea behind Kafka, Pulsar, and Kinesis: don't delete messages — keep a log and let readers track their own position.

---

## 🧒 ELI5 — Explain Like I'm 5

A normal postbox: you take a note out, and **it's gone**. Only one person ever reads it.

A **logbook** is different. Nobody takes anything out. Every note is **written into a book, in order, forever** (or for a set number of days).

Each reader keeps a **bookmark** saying which line they've got up to.

Look at what this changes:

- **Three different teams can read the same book** at their own speed, each with their own bookmark. Nobody blocks anyone.
- **You can move your bookmark backwards** and read last Tuesday again. Found a bug in how you counted desserts? Rewind and recount. **You cannot do this with a postbox** — those notes are gone.
- **A new team joins next year?** They start at line 1 and catch up. The other teams don't even notice.
- **Writing is very fast**, because you always write at the end of the book — never in the middle. (Writing at the end of a notebook is much faster than finding the right page.)

The costs:

- The book takes **space**, so you eventually tear out old pages. *(Retention.)*
- To let 10 people read in parallel you need **10 separate books** split by topic — and then "in order" only holds *within* each book. *(Partitions.)*

That's a log. Everything about Kafka follows from it.

---

## Queue vs log, precisely

|  | Traditional queue | Log |
|---|---|---|
| Storage | Message deleted on ack | Append-only, retained by time or size |
| Position | Broker tracks per-message state | **Consumer tracks an offset** |
| Consumers per message | One | Every consumer group |
| Replay | Impossible | Reset the offset |
| Ordering | Weak | Strict per partition |
| Broker state | Complex (per-message flags, redelivery timers) | Simple (a file and an index) |
| Throughput | Good | **Very high** — sequential I/O, zero-copy, batching |
| Adding a consumer later | It only sees new messages | It can read all of history |
| Parallelism limit | Number of consumers | **Number of partitions** |

🎯 **The architectural insight:** moving position-tracking from the broker to the consumer makes the broker nearly stateless per message. That's why a log scales to millions of messages per second on modest hardware — the broker is essentially appending to a file.

---

## The physical structure

```
Topic: orders
├── Partition 0 : [0][1][2][3][4][5][6]...        ← append here
├── Partition 1 : [0][1][2][3][4]...
└── Partition 2 : [0][1][2][3][4][5][6][7]...

Consumer group "billing"   : P0@offset 4, P1@offset 2, P2@offset 7
Consumer group "analytics" : P0@offset 6, P1@offset 4, P2@offset 7
                             ↑ independent bookmarks, same data
```

| Concept | Meaning |
|---|---|
| **Topic** | A named stream of records |
| **Partition** | An ordered, immutable sequence — the unit of parallelism *and* of ordering |
| **Offset** | A monotonically increasing position within a partition |
| **Consumer group** | A set of consumers sharing the work of a topic; each partition is assigned to exactly one member |
| **Retention** | How long records are kept (time, size, or compacted) |
| **Segment** | The on-disk files a partition is split into; deletion happens per segment |

### Why it's so fast

1. **Sequential writes.** Appending to a file is 100–1000× faster than random writes, even on SSD.
2. **Page cache.** Recently written data is still in OS memory, so consumers reading near the head never touch disk.
3. **Zero-copy** (`sendfile`). Data goes from page cache straight to the socket without passing through application memory.
4. **Batching + compression.** Producers batch records; brokers store the batch as-is; consumers decompress. Compression ratios of 5–10× are common on JSON.

🔢 A single modest broker sustains **hundreds of MB/s** and hundreds of thousands of messages per second because of these four properties combined.

---

## Partitions: the central design decision

**Partition count determines:**
- **Max consumer parallelism** in a group — you cannot have more useful consumers than partitions. Extra consumers sit idle.
- **Ordering scope** — order is guaranteed within a partition, never across.
- **Throughput ceiling** — each partition's writes go to one leader broker.

### Choosing the partition key

```
partition = hash(key) % num_partitions
```

| Key choice | Ordering guarantee | Risk |
|---|---|---|
| `user_id` | All events for a user, in order | A power user becomes a hot partition |
| `order_id` | Per-order ordering | Very even spread; no cross-order ordering |
| `tenant_id` | Per-tenant ordering | Large tenants skew badly |
| `null` (round robin) | None | Perfectly even; no ordering at all |
| A composite (`tenant:entity`) | Per-entity within tenant | Good compromise |

☠️ **Two failure modes to name:**

**Hot partition.** A skewed key sends most traffic to one partition, which is served by one broker and one consumer. Aggregate capacity looks fine; one partition is saturated. Mitigations: add a salt to the hot key and re-aggregate downstream; or use a composite key with higher cardinality.

**Repartitioning breaks ordering.** Going from 6 to 12 partitions changes `hash(key) % N` for most keys, so a key's history is split across two partitions and ordering is silently violated for records in flight. Mitigations: **over-provision partitions up front** (they're cheap-ish), or do a controlled migration — drain, stop producing, reassign, resume.

**How many?** Rules of thumb: enough for your target parallelism plus headroom (2–4× current consumer count), keep per-partition throughput under ~10 MB/s, and remember each partition costs file handles, memory, and leader-election time — tens of thousands of partitions per cluster is where operations get painful. **Start at 12–24 for a busy topic**, not 3.

---

## Consumer groups and rebalancing

```
Topic with 6 partitions
  Group A with 2 consumers → each gets 3 partitions
  Group A scaled to 6      → each gets 1
  Group A scaled to 10     → 6 work, 4 idle
```

**Rebalancing** happens when a consumer joins, leaves, or is deemed dead. During a classic (eager) rebalance, **all consumers stop** while partitions are reassigned — a stop-the-world pause.

| Mitigation | Effect |
|---|---|
| **Cooperative/incremental rebalancing** | Only the moving partitions pause |
| **Static membership** (`group.instance.id`) | A restarting consumer keeps its partitions; no rebalance on rolling deploys |
| Tune `session.timeout.ms` / `max.poll.interval.ms` | Prevents a slow consumer being wrongly evicted |

☠️ **The `max.poll.interval.ms` trap:** if processing a batch takes longer than this (default 5 min), the broker assumes the consumer is dead and rebalances — while the consumer is still working. The consumer then fails to commit, the work is redone by someone else, and this can loop forever. Fix: smaller `max.poll.records`, faster processing, or a longer interval.

---

## Offsets and delivery semantics

| Commit strategy | Guarantee | Failure behaviour |
|---|---|---|
| Commit **before** processing | At-most-once | Crash → message never processed |
| Commit **after** processing | At-least-once | Crash → reprocessed (duplicates) |
| Commit **with** the side effect, atomically | Effectively-once | Requires a transaction spanning both |

**Default: commit after processing, and make consumers idempotent.**

**Effectively-once** is achievable in two ways:
1. **Store the offset in the same database as the result.** `BEGIN; INSERT result; UPDATE offsets SET pos=N; COMMIT;` — on restart, read the offset from your own database, not from Kafka.
2. **Kafka transactions** — `read → process → write` all within Kafka, with `isolation.level=read_committed` on the downstream consumer. This is what "exactly-once semantics" means, and it does **not** extend to external side effects like sending an email.

---

## Retention and compaction

| Policy | Behaviour | Use |
|---|---|---|
| **Time-based** (`retention.ms`) | Delete segments older than N | Event streams (7 days is typical) |
| **Size-based** (`retention.bytes`) | Delete oldest when the partition exceeds N | Bounded disk |
| **Log compaction** (`cleanup.policy=compact`) | Keep the **latest value per key**, forever | Changelogs, current-state topics |
| **Compact + delete** | Both | Bounded changelogs |

**Log compaction is a genuinely different tool.** A compacted topic converges to "the current value for every key" — effectively a durable, replayable key-value snapshot. A new consumer replays it from offset 0 and materialises the complete current state. This underpins CDC pipelines, Kafka Streams state stores, and cache-warming patterns.

⚠️ Deleting a key in a compacted topic requires a **tombstone** — a record with the key and a `null` value. Compaction keeps tombstones for `delete.retention.ms` so all consumers see the deletion, then removes them.

---

## Replay: the capability nothing else gives you

| Scenario | Action |
|---|---|
| A consumer bug corrupted derived data | Fix, reset the offset to before the bug, reprocess |
| A new service needs historical context | Start a new consumer group at offset 0 |
| Testing a new algorithm | Run a shadow consumer group over real history |
| Rebuilding a search index or cache | Replay the compacted topic |
| Post-incident audit | Read exactly what was published, when |

```bash
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group billing --topic orders --reset-offsets --to-datetime 2026-08-30T00:00:00.000 \
  --execute
```

🎯 **In an interview:** *"I'd use a log rather than a queue here because three teams need the same events independently, and because being able to replay after a bug is worth the extra operational cost. With a plain queue, a bug in the analytics consumer would mean permanently lost data."* That is a complete, defensible justification.

---

## When a log is the wrong choice

| Situation | Why |
|---|---|
| Simple background jobs with one consumer | SQS is simpler and cheaper to operate |
| You need per-message priority | Logs have no priority — you need separate topics |
| You need per-message TTL or delay | Logs are position-based, not time-based per message |
| Very large payloads | Use a claim check; logs aren't blob storage |
| Tiny scale (< 100 msg/s) and a small team | A stateful distributed system for no benefit |
| Strict global ordering across all keys | Requires one partition = no parallelism |

⚖️ **The honest cost:** Kafka is a stateful distributed system with brokers, controllers (or ZooKeeper historically), partition rebalancing, disk management, and non-trivial client tuning. Managed offerings (MSK, Confluent Cloud, Redpanda Cloud) remove much of that at a price. Don't propose it without a reason from the list of things only a log can do.

---

## The alternatives compared

| System | Notes |
|---|---|
| **Kafka** | The reference implementation. Huge ecosystem (Connect, Streams, ksqlDB). KRaft mode removes ZooKeeper |
| **Pulsar** | Separates serving (brokers) from storage (BookKeeper) — faster rebalancing, easier scaling, native multi-tenancy and tiered storage |
| **Redpanda** | Kafka-API-compatible, C++, no JVM, thread-per-core; simpler ops, lower tail latency |
| **AWS Kinesis** | Managed; shard-based; 24 h–365 day retention; simpler but less flexible |
| **Redis Streams** | In-memory log; great for modest scale ([lab](04-redis-queue-tutorial.md)) |
| **Google Pub/Sub** | Managed; more pub/sub than log, but supports replay via snapshots and seek |

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| State the queue vs log differences across eight dimensions | ☐ |
| Explain why moving offsets to the consumer makes the broker fast | ☐ |
| Name the four reasons logs achieve high throughput | ☐ |
| Explain what partition count determines (three things) | ☐ |
| Describe hot partitions and repartitioning-breaks-ordering | ☐ |
| Explain rebalancing and three mitigations | ☐ |
| Explain the `max.poll.interval.ms` trap | ☐ |
| Give two ways to achieve effectively-once | ☐ |
| Explain log compaction and tombstones | ☐ |
| Give five replay scenarios | ☐ |
| Name six situations where a log is the wrong tool | ☐ |

---

**← Previous** [2.2.4 Redis Queue Tutorial](04-redis-queue-tutorial.md)
**Next →** [2.2.6 Introduction to Kafka](06-introduction-to-kafka.md)
