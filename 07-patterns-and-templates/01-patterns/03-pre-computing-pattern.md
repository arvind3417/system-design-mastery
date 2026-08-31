# 7.1.3 — Pre-Computing Pattern

> **Part 7 · Patterns & Templates · Patterns · Chapter 3 of 10**
> Do the work before it's asked for. The most reliable way to make reads fast.

---

## 🧒 ELI5 — Explain Like I'm 5

Someone will ask you a hard sum tomorrow: *"what's 8,432 × 917?"*

**Two options:**

1. **Wait for the question, then work it out.** They stand there for a minute while you do long multiplication. If a hundred people ask, you do it a hundred times.
2. **Work it out tonight and write the answer on a card.** Tomorrow you hand over the card instantly. A hundred people? Same card, a hundred times.

Option 2 is obviously better — **if** you can guess which sums will be asked.

And that's the whole catch. You can't precompute *every possible* sum; there are infinitely many. So you have to work out **which questions actually get asked**, and precompute those.

The three things that decide whether it's worth it:

- **How often is it asked?** Once → not worth it. A million times → absolutely.
- **How expensive is the sum?** Easy → don't bother. Hard → definitely.
- **How many different sums are there?** A hundred → precompute all of them. A trillion → you can't.

---

## The trade

$$\text{compute on read} = R \times C \qquad\qquad \text{precompute} = W \times C \;+\; \text{storage}$$

Precomputing wins when **R ≫ W** — which is most read-heavy systems.

| | Compute on read | Precompute |
|---|---|---|
| Read latency | ❌ Full cost | ✅ A lookup |
| Write cost | ✅ None | Recompute on change |
| Storage | ✅ None | One entry per result |
| Freshness | ✅ Always exact | Bounded staleness |
| Flexibility | ✅ Any query | Only precomputed shapes |
| Wasted work | None | If never read |

🎯 **The same trade as [push vs pull](../../03-scaling-services/04-dataflow/02-push-vs-pull.md), [materialized views](../../06-big-data/06-realtime-and-analytics/01-materialized-views-streaming.md), and [caching](../../03-scaling-services/03-caching/01-caching-mental-model.md).** Recognising that these are one idea in different clothing is worth saying out loud — it's a sign you're reasoning from principles rather than reciting patterns.

---

## The three questions

Before precomputing anything:

| Question | Precompute if |
|---|---|
| **1. What's the read:write ratio?** | ≫ 1 |
| **2. How expensive is the computation?** | Expensive relative to a lookup |
| **3. What's the cardinality of the result space?** | Bounded and enumerable |

☠️ **Question 3 is the one that kills otherwise-good ideas.** Precomputing "search results for every possible query" is impossible — the query space is unbounded. Precomputing "the top 100 products per category per region" is 200 categories × 20 regions = 4,000 entries. **Trivial.**

**When cardinality is too high, precompute the *components* and compose at read time:** precompute per-product scores (bounded), and combine them per query (cheap). That hybrid is usually the answer.

---

## What to precompute

| Candidate | Read:write | Cost | Cardinality | Verdict |
|---|---|---|---|---|
| User timeline | 1000:1 | High (fan-in of N sources) | 1 per user | ✅ Precompute |
| Follower count | 1000:1 | High (`COUNT` on a big table) | 1 per user | ✅ Precompute |
| Product page | 100:1 | Medium (several joins) | 1 per product | ✅ Precompute |
| Leaderboard | 10000:1 | High (global sort) | Few | ✅ Precompute |
| Recommendations | 100:1 | Very high (ML) | 1 per user | ✅ Precompute (batch) |
| Daily revenue report | 100:1 | Very high (scan) | 1 per day | ✅ Precompute |
| Search results | 1:1 varied | Medium | ❌ Unbounded | ❌ Index instead |
| "Orders between two arbitrary dates" | 1:1 | Low | ❌ Unbounded pairs | ❌ Query it |
| A user's exact current balance | 10:1 | Low | 1 per user | ❌ Just read it — and it must be exact |

---

## The mechanisms

### 1. Materialised on write
```python
def post_tweet(author, text):
    tid = tweets.insert(author, text)
    for follower in followers(author):                # precompute at write time
        redis.lpush(f"timeline:{follower}", tid)
```
✅ Freshest. ❌ Write cost scales with fan-out — see [Push vs Pull](../../03-scaling-services/04-dataflow/03-push-vs-pull-twitter-timeline.md) for the celebrity problem.

### 2. Incremental (streaming)
```sql
INSERT INTO revenue_by_region
SELECT region, window_start, SUM(amount) FROM TABLE(TUMBLE(...)) GROUP BY ...;
```
✅ Amortised — the work happens once per event, not once per read. The standard modern approach.

### 3. Scheduled batch
```sql
-- nightly
CREATE TABLE recommendations AS SELECT ... FROM expensive_ml_pipeline;
```
✅ Cheapest for expensive computations tolerant of staleness. Recommendations, rankings, reports.

### 4. Lazy with caching
Compute on first request, cache the result. ✅ Only pays for what's actually read — the right choice when the result space is large but the *accessed* subset is small.

🎯 **Choose by the access pattern:** if nearly everything is read, precompute eagerly. If a small unpredictable subset is read, compute lazily and cache. **Eagerly precomputing 10 million timelines when 200,000 users log in daily wastes 98% of the work** — which is why real systems fan out only to *active* users.

---

## Multi-level precomputation

```
raw events
  → per-minute aggregates      (fine, short retention)
    → per-hour                 (built from minutes, not raw)
      → per-day                (built from hours)
        → per-month            (built from days)
```

🔢 Computing a monthly total from 30 daily rows rather than 2.6 billion raw events is **eight orders of magnitude** cheaper — and each level is built from the one below, so the raw data is scanned exactly once.

⚠️ **Only works for aggregations that compose.** Sums and counts do. **Distinct counts and percentiles do not** — you must store the mergeable sketch (HyperLogLog, t-digest) at each level, not the final number. **Store the intermediate, not the answer**, and you keep the ability to roll up.

---

## Keeping it fresh

| Strategy | Freshness | Cost |
|---|---|---|
| Recompute on every write | ✅ Exact | High write cost |
| Incremental update on write | ✅ Near-exact | ✅ Low |
| Scheduled refresh | Interval-bounded | ✅ Low |
| Refresh on read if stale | Bounded | Lazy |
| ✅ **Incremental + periodic full rebuild** | Near-exact, self-correcting | Moderate |

🎯 **The hybrid is the production answer**, and it recurs throughout this book: incremental updates keep it fresh; a periodic full rebuild corrects drift from bugs, missed events, and restarts. It converts an unbounded correctness risk into a bounded one, cheaply.

⚠️ **Incremental updates drift.** A missed event, a non-idempotent increment applied twice, a bug in the delta logic — each leaves a permanent error, because nothing ever recomputes from scratch. **Always have the rebuild path, and run it.**

---

## Precomputation with personalisation

The hard case: the result depends on *who is asking*, so the cardinality is users × items.

| Technique | How |
|---|---|
| **Precompute the shared part** | Global rankings, item features, popularity |
| **Compute the personal part at read** | Filter and re-rank a precomputed candidate set |
| **Precompute per segment, not per user** | 50 segments instead of 50 million users |
| **Precompute for active users only** | ✅ 10–20% of the user base |
| **Two-stage: retrieve then rank** | Precompute a candidate set of ~500; rank per request |

🎯 **Two-stage retrieval is how every large recommendation and search system works**, and it's a strong thing to name: an offline stage narrows billions of items to a few hundred candidates (precomputed, cheap), and an online stage ranks those few hundred per request (personalised, fast). Neither stage alone is tractable; together they are.

---

## The costs

| Cost | Detail |
|---|---|
| **Storage** | One entry per result. 200M users × 800 IDs × 8 B ≈ 1.3 TB |
| **Write amplification** | One write triggers N precomputation updates |
| **Staleness** | Results are as fresh as the last update |
| **Wasted computation** | Precomputing for users who never return |
| **Complexity** | A pipeline to build, monitor, and rebuild |
| **Rigidity** | ☠️ A new query shape needs a new precomputation **and a backfill** |

☠️ **The rigidity cost is the one that hurts later.** Precomputing "top products by category" is fine — until product wants "top products by category *and price band*", which no existing precomputation answers. You now need a new pipeline plus a historical backfill. **Precomputation trades flexibility for speed**, and that bill arrives months after the decision.

⚖️ **Mitigation: precompute at a finer granularity than you currently need**, so more queries can be answered by aggregating up. Storing per-(product, day, region) lets you answer per-product, per-day, per-region, and any combination — for slightly more storage and far more flexibility.

---

## When not to precompute

| Situation | Why |
|---|---|
| Reads ≈ writes | No amortisation |
| The computation is cheap | Lookup ≈ compute |
| Unbounded result space | Cannot enumerate |
| The result must be exact and current | Balances, inventory at checkout |
| Requirements are still changing | You'd rebuild the pipeline repeatedly |
| Most results are never read | Wasted work — go lazy |

---

## The interview framing

> "Timeline reads outnumber writes about 1000 to 1, and building a timeline on read means a fan-in across 200 followees — so I'd precompute it on write into a per-user list, capped at 800 entries. That's about 1.3 TB of IDs across 200 million users, which fits in a Redis cluster because I'm storing IDs, not content.
>
> Two refinements: I'd fan out only to users active in the last 30 days, which cuts the work about five-fold and rebuilds lazily for returning users; and I'd skip fan-out entirely above a follower threshold, pulling those at read time, so no single write causes 100 million updates.
>
> The costs I'm accepting: seconds of staleness, storage for the precomputed lists, and rigidity — if the product later wants a differently-ranked feed, that's a new precomputation plus a backfill. To bound the drift risk, incremental updates run continuously and a periodic rebuild corrects them."

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| State the read-vs-write cost trade | ☐ |
| Ask the three questions, especially cardinality | ☐ |
| Give the compose-at-read-time fallback for high cardinality | ☐ |
| Compare four precomputation mechanisms | ☐ |
| Explain why to precompute only for active users | ☐ |
| Explain multi-level rollups and why to store sketches | ☐ |
| Recommend incremental + periodic rebuild, and say why | ☐ |
| Explain two-stage retrieval for personalisation | ☐ |
| Name the rigidity cost and its mitigation | ☐ |
| Deliver the interview framing | ☐ |

---

**← Previous** [7.1.2 Cache-First Pattern](02-cache-always.md)
**Next →** [7.1.4 Database Per Microservice](04-database-per-service.md)
