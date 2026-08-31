# 6.3.2 — Windowing Patterns

> **Part 6 · Big Data · Stream Processing · Chapter 2 of 5**
> You can't aggregate an infinite stream. Windows cut it into finite pieces.

---

## 🧒 ELI5 — Explain Like I'm 5

You can't add up a list that never ends. So you **cut it into chunks** and add up each chunk. The only question is *how* you cut.

**Way 1 — the egg box.** Every 10 minutes is one compartment. 10:00–10:10, 10:10–10:20. Every sale goes in exactly one compartment. Simple, tidy, no overlaps. *(Tumbling windows.)*

**Way 2 — the moving spotlight.** "How many sales in the **last** 10 minutes?" — asked every minute. So the 10-minute spotlight slides forward, and each sale is caught by **ten different** spotlights. Ten times the work, but the answer is always current. *(Sliding windows.)*

**Way 3 — natural pauses.** A shopper picks up things for a while, then leaves. You don't want fixed 10-minute chunks; you want "**everything they did before they wandered off for 30 minutes**." The chunk ends itself when nothing happens for a while — so chunks are all different lengths. *(Session windows.)*

Which one you pick isn't a technical detail. **It's the question you're asking**, expressed in time.

---

## The three window types

```
Events:  ●   ● ●     ●  ●    ●   ●●      ●

TUMBLING (5 min):  [────][────][────][────]
                   no overlap, no gaps, each event in exactly ONE window

SLIDING (10 min window, 5 min slide):
                   [─────────]
                        [─────────]
                             [─────────]
                   overlapping — each event in TWO windows

SESSION (5 min gap):
                   [──]  [───────]      [─]
                   boundaries determined by the DATA, not the clock
```

| Type | Defined by | Event appears in | Use for |
|---|---|---|---|
| **Tumbling** | Size | Exactly 1 window | Periodic reports: hourly revenue, daily active users |
| **Sliding (hopping)** | Size + slide | `size / slide` windows | Moving averages, "last N minutes" alerts |
| **Session** | Inactivity gap | 1 dynamic window | User sessions, device activity bursts |
| **Global** | Nothing (custom trigger) | 1 unbounded window | Running totals with a custom emit rule |

---

## Tumbling windows

```java
stream.keyBy(e -> e.userId)
      .window(TumblingEventTimeWindows.of(Time.minutes(5)))
      .aggregate(new SumAggregator());
```

| ✅ | ❌ |
|---|---|
| Simplest; each event counted once | Boundary effects — a burst spanning two windows looks like two small bursts |
| ✅ Minimal state — one accumulator per key per window | Fixed alignment may not match business hours or time zones |
| Results are non-overlapping and easy to store | |

⚠️ **Boundary effects matter more than people expect.** A traffic spike from 10:04 to 10:06 is split across the 10:00–10:05 and 10:05–10:10 windows, so a "spike detector" thresholding on window totals may see two moderate windows and miss it entirely. Sliding windows exist largely to fix this.

⚠️ **Alignment and time zones:** a "daily" tumbling window aligned to UTC is not a business day in Tokyo. Flink and Spark support an offset; set it deliberately.

---

## Sliding windows

```java
.window(SlidingEventTimeWindows.of(Time.minutes(10), Time.minutes(1)))
// 10-minute window, emitted every minute
```

🔢 **The cost multiplier:** each event belongs to `size / slide` windows. A 1-hour window sliding every 1 minute means **every event is stored and aggregated in 60 windows** — 60× the state and 60× the aggregation work.

☠️ **This is the classic streaming cost blow-up.** A 24-hour window sliding every minute puts each event in **1,440** windows. At 100,000 events/second that is 144 million window-entries per second of work. The job will not survive.

**Mitigations:**

| Technique | Effect |
|---|---|
| **Larger slide** | 1-hour window sliding every 5 min = 12 windows, not 60 |
| **Incremental aggregation** | Store an accumulator, not the raw events (`AggregateFunction`, not `ProcessWindowFunction` over an iterable) |
| **Tumbling sub-windows + roll-up** | Compute 1-minute tumbling windows, then sum the last 60 at query time — **state is 1×, not 60×** |
| **Approximate structures** | HyperLogLog for distinct counts, t-digest for percentiles |

🎯 **The tumbling-plus-roll-up trick is the professional answer** to "we need a 24-hour rolling metric." Compute small tumbling windows once, store them, and combine at read time. It converts a 1,440× state problem into a 1× one.

---

## Session windows

```java
.window(EventTimeSessionWindows.withGap(Time.minutes(30)))
```

Windows begin with an event and end after a **gap of inactivity**. Length is data-dependent.

```
user A:  ●●●  ──30min gap──  ●●     →  two sessions
user B:  ●   ●   ●   ●   ●          →  one long session
```

| ✅ | ❌ |
|---|---|
| Matches real user behaviour | Windows must **merge** when a late event bridges two of them — complex |
| Naturally variable duration | State per key until the gap elapses |
| ✅ The right model for engagement analytics | An always-active user's session never closes |

⚠️ **Session merging is the subtle part.** Events at 10:00 and 10:40 create two sessions (30-minute gap). If an event for 10:20 then arrives late, those two sessions must **merge into one**. Flink supports this; it requires mergeable window state, which restricts which aggregations you can use.

⚠️ **The never-ending session:** a bot or an always-on device produces a session that never closes and whose state grows forever. **Cap session duration** (e.g. force-close after 4 hours) as a safety valve.

---

## Windows and state size

$$\text{state} \approx \text{keys} \times \text{windows per key} \times \text{accumulator size}$$

```
10M users × 1 tumbling 5-min window × 100 B          =    1 GB   ✅
10M users × 60 sliding windows × 100 B               =   60 GB   ⚠️
10M users × 1,440 sliding windows × 100 B            = 1.44 TB   ❌
```

**Reducing state:**

| Technique | Effect |
|---|---|
| **Incremental aggregation** | Keep a running sum, not the event list. Often 100–1000× smaller |
| **Fewer, larger slides** | Linear reduction |
| **State TTL** | Expire keys that stop appearing |
| **Approximate sketches** | HLL: millions of distinct values in ~12 KB |
| **Pre-aggregate before `keyBy`** | Local combining, like a MapReduce combiner |

🎯 **`AggregateFunction` vs `ProcessWindowFunction` is the single most important Flink API choice.** The first keeps one accumulator per window; the second buffers **every event** so it can iterate them at the end. The latter is often written by accident and multiplies state by the number of events per window.

```java
// ✅ incremental — constant state per window
.aggregate(new SumAggregator())

// ❌ buffers every event in the window
.process(new ProcessWindowFunction<>() { ... })

// ✅ both: incremental aggregation, plus window metadata at emit time
.aggregate(new SumAggregator(), new AddWindowMetadata())
```

---

## Triggers: when results are emitted

A window defines *what* is aggregated; a **trigger** defines *when* the result is emitted.

| Trigger | Behaviour |
|---|---|
| **On watermark** (default) | Emit once, when the window is provably complete |
| **Early firing** | Emit partial results every N seconds while the window is open |
| **Late firing** | Emit an update when a late event arrives |
| **Count-based** | Emit every N events |
| **Custom** | Any combination |

```java
.window(TumblingEventTimeWindows.of(Time.hours(1)))
.trigger(ContinuousEventTimeTrigger.of(Time.minutes(1)))   // early updates
.allowedLateness(Time.minutes(10))                          // late corrections
```

🎯 **Early triggers are how you get a live dashboard from a long window.** An hourly window that only emits at the end gives you a number once an hour; with a 1-minute early trigger you see the hour building in real time, and the final value at the watermark. **The cost is that downstream must handle updates to a result it already received.**

**How refinements combine** — the "how" of the Dataflow model:

| Mode | Behaviour | Downstream requirement |
|---|---|---|
| **Accumulating** | Each emission is the full result so far | Overwrite by key (upsert) ✅ simplest |
| **Discarding** | Each emission is only the delta | Downstream must add — and must not lose one |
| **Accumulating + retracting** | Emits a retraction of the old value plus the new | Needed for exact joins downstream |

⚠️ **Choose accumulating with an upsert sink unless you have a reason not to.** It's idempotent, so duplicate emissions are harmless — which composes well with at-least-once delivery.

---

## Choosing a window

| Question | Window |
|---|---|
| "Revenue per hour" | Tumbling 1 h |
| "Alert if errors exceed 100 in the last 5 minutes" | Sliding 5 min / 30 s |
| "Average session length" | Session, 30 min gap |
| "Daily active users" | Tumbling 1 day (+ HLL for distinct count) |
| "Moving average over 7 days, updated hourly" | ✅ Tumbling 1 h + roll-up at query time |
| "Unique visitors in the last hour, updated per second" | Sliding + HLL, or tumbling 1 min + roll-up |
| "Detect 3 failures then a success" | Pattern matching (CEP), not a window |

⚠️ **If the question contains "in the last N", you probably want sliding — or, better, tumbling with a roll-up.** If it contains "per N", you want tumbling. If it contains "per session" or "per visit", you want session windows.

---

## Windows in SQL

```sql
-- Flink SQL
SELECT window_start, window_end, user_id, SUM(amount)
FROM TABLE(TUMBLE(TABLE orders, DESCRIPTOR(order_time), INTERVAL '5' MINUTES))
GROUP BY window_start, window_end, user_id;

SELECT window_start, COUNT(*)
FROM TABLE(HOP(TABLE clicks, DESCRIPTOR(click_time),
               INTERVAL '1' MINUTE, INTERVAL '10' MINUTES))
GROUP BY window_start;

SELECT user_id, SESSION_START(ts, INTERVAL '30' MINUTE), COUNT(*)
FROM events GROUP BY user_id, SESSION(ts, INTERVAL '30' MINUTE);
```

Note `TUMBLE`/`HOP`/`SESSION` and the explicit **time attribute descriptor** — the SQL must say which column is event time, because that's the whole basis of the windowing.

---

## ☠️ Failure modes

| Mistake | Consequence |
|---|---|
| Sliding window with a tiny slide | State and CPU multiply by `size/slide`; the job dies |
| `ProcessWindowFunction` where `AggregateFunction` would do | State proportional to event count |
| No state TTL on keyed windows | Unbounded growth as the key space grows |
| Session windows with no maximum duration | A bot's session never closes |
| Processing-time windows | Non-deterministic; replays give different answers |
| Ignoring late data | Silently wrong results ([Event Time & Watermarks](03-event-time-watermarks.md)) |
| Discarding-mode output into a non-transactional sink | A lost delta is permanently wrong |
| Tumbling windows aligned to UTC for a business-day metric | Off-by-one-day errors |

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| Draw all three window types | ☐ |
| Explain boundary effects in tumbling windows | ☐ |
| Compute the sliding-window cost multiplier | ☐ |
| Give the tumbling-plus-roll-up alternative | ☐ |
| Explain session merging and the never-ending session | ☐ |
| Estimate window state from keys × windows × accumulator | ☐ |
| Explain `AggregateFunction` vs `ProcessWindowFunction` | ☐ |
| Explain triggers and early firing | ☐ |
| Name the three refinement modes and pick the default | ☐ |
| Choose a window type from a business question | ☐ |

---

**← Previous** [6.3.1 What Is Stream Processing?](01-stream-processing-intro.md)
**Next →** [6.3.3 Event Time & Watermarks](03-event-time-watermarks.md)
