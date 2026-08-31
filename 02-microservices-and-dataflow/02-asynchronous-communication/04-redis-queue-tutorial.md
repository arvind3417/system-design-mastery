# 2.2.4 — Redis Queue Tutorial

> **Part 2 · Microservices & Data Flow · Asynchronous Communication · Chapter 4 of 7**
> 🧪 **Hands-on lab.** Build a queue three ways in Redis, break it, then fix it. Run the commands.

---

## 🧒 ELI5 — Explain Like I'm 5

Redis is a very fast box of things. You can build a postbox out of it in three ways, each better than the last:

1. **A simple pile of notes.** Push notes on one end, take from the other. Easy! But if a cook takes a note and then trips over, **the note is gone forever** — nobody knows it existed.
2. **A pile plus a "being worked on" shelf.** When you take a note you *immediately* put a copy on the shelf. If you finish, you remove it from the shelf. If you vanish, someone sees an old note sitting on the shelf and puts it back in the pile. **Nothing is lost.**
3. **A proper logbook with names.** Every note is written in a book that remembers who took it, when, and how many times it's been tried. Multiple teams can each read the whole book at their own pace. This is Redis's built-in version, and it's the one to use.

We'll build all three, because the *reason* version 3 exists only makes sense once you've been bitten by version 1.

---

## Setup

```bash
docker run -d --name redis-lab -p 6379:6379 redis:7-alpine
docker exec -it redis-lab redis-cli
```

---

## Version 1 — the naive list queue (and why it loses data)

```bash
# Producer
LPUSH jobs '{"id":"1","task":"resize","img":"a.jpg"}'
LPUSH jobs '{"id":"2","task":"resize","img":"b.jpg"}'

# Consumer (blocking pop, waits up to 5s)
BRPOP jobs 5
# 1) "jobs"
# 2) "{\"id\":\"2\",...}"   ← wait, that's wrong order?
```

⚠️ `LPUSH` + `BRPOP` gives FIFO (push left, pop right). `LPUSH` + `BLPOP` gives LIFO. Getting this backwards is a classic bug — **check which end you're using.**

### 🧪 Drill 1 — prove the data loss

```bash
# Terminal A: consumer takes a job then "crashes"
BRPOP jobs 0
# (you now hold the message)
# Ctrl-C — simulate the worker dying before finishing

# Terminal B
LLEN jobs
# → the message is NOT in the queue. It is gone. Permanently.
```

**Result: at-most-once delivery.** The message was removed the instant it was read. This is fine for metrics and disposable telemetry, and unacceptable for anything that matters.

---

## Version 2 — reliable queue with `BLMOVE`

Move the message **atomically** from the pending list to a per-worker processing list.

```bash
# Consumer: atomically pop from `jobs` and push to `processing:worker1`
BLMOVE jobs processing:worker1 RIGHT LEFT 5

# The message now exists in BOTH conceptually-safe places:
LRANGE processing:worker1 0 -1

# On success, remove it from processing
LREM processing:worker1 1 '{"id":"1",...}'
```

### 🧪 Drill 2 — prove recovery works

```bash
BLMOVE jobs processing:worker1 RIGHT LEFT 0
# Ctrl-C (crash)

LRANGE processing:worker1 0 -1
# → the message is still there. Nothing was lost.

# A janitor process returns it:
RPOPLPUSH processing:worker1 jobs
```

**Result: at-least-once delivery.** Better. But look at what you now have to build yourself:

| Missing piece | You must implement |
|---|---|
| Detecting a dead worker | Heartbeats + a janitor scanning processing lists |
| Retry counting | A separate `HINCRBY retries:{id}` |
| Dead-letter | Manual: after N retries, `LPUSH dlq` |
| Multiple independent consumer groups | Not possible — one list, one set of consumers |
| Message age / lag visibility | Manual timestamps |

☠️ **The janitor is harder than it looks.** How do you know `worker1` is dead and not just slow? You need worker heartbeats in a separate key with a TTL, and a scanner that reclaims lists whose heartbeat has expired. This is exactly the machinery Redis Streams provides.

---

## Version 3 — Redis Streams (use this)

Streams are an append-only log with **consumer groups**, per-message delivery tracking, and a pending-entries list. This is the right tool.

```bash
# ── Producer ────────────────────────────────────────────
XADD jobs '*' task resize img a.jpg
XADD jobs '*' task resize img b.jpg
# '*' = auto-generate the ID (timestamp-sequence)

# ── Create a consumer group starting at the beginning ───
XGROUP CREATE jobs workers 0 MKSTREAM

# ── Consumer reads as a named member of the group ───────
XREADGROUP GROUP workers worker1 COUNT 10 BLOCK 5000 STREAMS jobs '>'
# '>' means "messages never delivered to this group"

# ── Acknowledge when done ───────────────────────────────
XACK jobs workers 1756633451000-0
```

### The Pending Entries List (PEL) — the important part

Every message delivered but not yet `XACK`ed sits in the group's PEL, with **who has it** and **how long they've had it**.

```bash
XPENDING jobs workers
# 1) (integer) 2                       ← count pending
# 2) "1756633451000-0"                 ← min ID
# 3) "1756633452000-0"                 ← max ID
# 4) 1) 1) "worker1"
#       2) "2"                         ← worker1 holds 2

# Detailed view with idle time
XPENDING jobs workers - + 10
# → id, consumer, idle_ms, delivery_count
```

### Claiming abandoned messages

```bash
# Reclaim anything idle for more than 60s (worker1 probably died)
XAUTOCLAIM jobs workers worker2 60000 0 COUNT 10
```

That one command replaces the entire janitor you would have written for version 2. It is **the reason to use Streams.**

### 🧪 Drill 3 — crash and recover with Streams

```bash
# Worker 1 reads but never acks
XREADGROUP GROUP workers worker1 COUNT 1 STREAMS jobs '>'
# (simulate crash — just don't ack)

XPENDING jobs workers - + 10
# → shows worker1 holding it, idle time climbing

# Worker 2 takes over after 10 seconds of idleness
XAUTOCLAIM jobs workers worker2 10000 0
# → worker2 now owns it and can process it
```

### 🧪 Drill 4 — build a dead-letter queue

```bash
# Find messages delivered more than 5 times
XPENDING jobs workers - + 100
# inspect delivery_count; for any > 5:

XADD jobs-dlq '*' original_id "1756633451000-0" reason "max_retries_exceeded"
XACK jobs workers 1756633451000-0      # remove from the PEL
```

In application code, this is a loop that runs every few seconds. **Streams do not dead-letter for you** — the `delivery_count` is provided, and acting on it is yours.

### 🧪 Drill 5 — two independent consumer groups

```bash
XGROUP CREATE jobs analytics 0
XREADGROUP GROUP analytics an1 COUNT 10 STREAMS jobs '>'
# → analytics sees EVERY message, independently of `workers`
```

This is the log/pub-sub property that a plain list cannot give you: two teams, two offsets, one stream.

---

## Trimming — Streams grow forever otherwise

```bash
# Approximate trim to ~1M entries (fast — trims whole nodes)
XADD jobs MAXLEN '~' 1000000 '*' task resize img c.jpg

# Or trim by time (keep 24h)
XTRIM jobs MINID '~' 1756547051000
```

☠️ **Forgetting to trim is the #1 Redis Streams production incident.** Redis is in-memory; an untrimmed stream eventually consumes all RAM and triggers eviction or OOM. Always set `MAXLEN ~` on `XADD`, and monitor `XLEN`.

⚠️ Trimming removes entries **even if they are still pending** in some group's PEL. Trim conservatively relative to your worst-case consumer lag.

---

## Monitoring commands

```bash
XLEN jobs                              # stream length
XINFO STREAM jobs                      # first/last IDs, length, groups
XINFO GROUPS jobs                      # per group: consumers, pending, lag
XINFO CONSUMERS jobs workers           # per consumer: pending, idle
```

`XINFO GROUPS` reports **`lag`** — how many entries the group hasn't read. That is your queue-depth equivalent and what you should autoscale and alert on.

---

## Reference consumer (Python)

```python
import redis, json, time

r = redis.Redis(decode_responses=True)
STREAM, GROUP, CONSUMER = "jobs", "workers", "worker-1"
MAX_DELIVERIES, CLAIM_IDLE_MS = 5, 60_000

try:
    r.xgroup_create(STREAM, GROUP, id="0", mkstream=True)
except redis.ResponseError:
    pass                                   # group already exists

def handle(msg_id, fields):
    # MUST be idempotent — msg_id is a natural dedupe key
    if r.set(f"done:{msg_id}", 1, nx=True, ex=86400) is None:
        return                             # already processed
    do_work(fields)

while True:
    # 1. reclaim anything abandoned by a dead worker
    _, claimed, _ = r.xautoclaim(STREAM, GROUP, CONSUMER,
                                 min_idle_time=CLAIM_IDLE_MS, start_id="0-0",
                                 count=10)
    # 2. read new messages
    resp = r.xreadgroup(GROUP, CONSUMER, {STREAM: ">"}, count=10, block=5000) or []
    batch = claimed + [m for _, msgs in resp for m in msgs]

    for msg_id, fields in batch:
        info = r.xpending_range(STREAM, GROUP, min=msg_id, max=msg_id, count=1)
        deliveries = info[0]["times_delivered"] if info else 1

        if deliveries > MAX_DELIVERIES:
            r.xadd(f"{STREAM}-dlq", {"orig_id": msg_id, **fields})
            r.xack(STREAM, GROUP, msg_id)
            continue
        try:
            handle(msg_id, fields)
            r.xack(STREAM, GROUP, msg_id)
        except TransientError:
            pass                           # leave pending → redelivered later
        except PermanentError as e:
            r.xadd(f"{STREAM}-dlq", {"orig_id": msg_id, "err": str(e), **fields})
            r.xack(STREAM, GROUP, msg_id)
```

Every production concern from [Chapter 2.2.2](02-message-queues.md) is visible in ~35 lines: reclaim, retry counting, DLQ, idempotency, transient-vs-permanent classification.

---

## When Redis Streams is the right choice — and when it isn't

| ✅ Good fit | ❌ Reach for Kafka/SQS instead |
|---|---|
| You already run Redis | Retention beyond what fits in RAM |
| Throughput up to ~100k msg/s | Multi-day or indefinite replay of large volumes |
| Latency-sensitive (sub-millisecond) | You need strong durability guarantees for financial data |
| Modest retention (hours to a day) | Cross-datacenter replication |
| You want one system, not two | You need Kafka's ecosystem (Connect, Streams, ksqlDB) |

⚠️ **The durability caveat, stated plainly:** Redis persistence is AOF (`appendfsync everysec` by default — up to 1 second of writes can be lost) or RDB snapshots (much larger windows). Replication is **asynchronous**, so a failover can lose recent writes. For a job queue where a lost job means a missing thumbnail, that's acceptable. For payments, it is not — use a system with synchronous replication, or write to your database first and use the outbox pattern.

---

## 📋 Lab checklist

| Task | Done? |
|---|---|
| Ran drill 1 and observed message loss with `BRPOP` | ☐ |
| Ran drill 2 and recovered a message with `BLMOVE` | ☐ |
| Explained why the version-2 janitor is hard | ☐ |
| Created a stream, group, and consumer; acked a message | ☐ |
| Inspected the PEL with `XPENDING` | ☐ |
| Reclaimed an abandoned message with `XAUTOCLAIM` | ☐ |
| Built a DLQ path using `delivery_count` | ☐ |
| Created two independent consumer groups | ☐ |
| Set `MAXLEN ~` trimming and explained the OOM risk | ☐ |
| Stated the Redis durability caveat | ☐ |

---

**← Previous** [2.2.3 Message Queue Patterns](03-message-queue-patterns.md)
**Next →** [2.2.5 Log-based Message Queues](05-log-based-message-queues.md)
