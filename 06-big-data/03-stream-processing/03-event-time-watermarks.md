# 6.3.3 — Event Time & Watermarks

> **Part 6 · Big Data · Stream Processing · Chapter 3 of 5**
> The hardest idea in streaming, and the one that makes results correct and reproducible.

---

## 🧒 ELI5 — Explain Like I'm 5

You're counting how many people came into the shop **each hour**.

Easy — except the door counter is unreliable. Sometimes a person walks in at 10:55 but their beep doesn't reach you until **11:04**, because the wire was loose.

So at 11:00, when it's time to announce the 10:00–11:00 total, you face a genuine dilemma:

- **Announce immediately** → you'll be wrong, because more 10-o'clock beeps are still coming.
- **Wait forever** → you never announce anything.

So you make a **judgement call**: *"I'll wait 5 extra minutes. If a beep hasn't reached me by 11:05, I'm not counting it."*

That decision — **"I now believe I have everything up to 11:00"** — is a **watermark**.

It's a promise, not a fact. Sometimes a beep arrives at 11:09 and you were wrong. Then you choose: ignore it, or go back and correct the total.

**Wait longer → more accurate, slower. Wait less → faster, more wrong.** There is no third option. That trade-off is the whole chapter.

---

## Three notions of time

| Time | When | Stable across replays? |
|---|---|---|
| **Event time** | When it actually happened (embedded in the event) | ✅ Yes |
| **Ingestion time** | When it entered the system | ❌ No |
| **Processing time** | When the operator handled it | ❌ No |

```
Event occurred:   10:00:00   ← event time
Reached Kafka:    10:04:30   ← ingestion time
Processed:        10:04:31   ← processing time
```

🎯 **Use event time.** It is the only choice that makes results **deterministic** — replay the same input tomorrow and get the same output. Processing-time results depend on when you happened to run the job, which makes reprocessing, testing, and audits meaningless.

⚖️ Processing time is legitimate for: latency monitoring, rate limiting, and cases where "what the system saw when" is genuinely the question. Never for business aggregations.

---

## Why events arrive late

| Cause | Typical delay |
|---|---|
| Mobile device offline (tunnel, plane, flat battery) | Seconds to **days** |
| Network retries and buffering | Seconds |
| Kafka partition rebalance or consumer lag | Seconds to minutes |
| Batch upload from an edge device | Minutes to hours |
| Clock skew on the producer | ± minutes |
| A backfill of historical data | Arbitrary |

🔢 **Real-world lateness distributions are long-tailed:** p50 might be 200 ms, p99 30 seconds, and p99.99 several hours. **You cannot wait for the maximum.** You pick a point on the distribution and accept the rest as late.

---

## Watermarks

> **A watermark of time T is an assertion: "I believe I have seen all events with event time ≤ T."**

```
Stream (by processing order):  ●10:03  ●10:01  ●10:04  ●10:02  ●10:06
Watermark (max seen − 2 min):   10:01   10:01   10:02   10:02   10:04
                                                          ↑
                          when the watermark passes 10:05, the
                          10:00-10:05 window fires
```

The watermark **drives window firing**. A window closes and emits when the watermark passes its end.

### Generation strategies

```java
// Bounded out-of-orderness: the most common
WatermarkStrategy.<Event>forBoundedOutOfOrderness(Duration.ofSeconds(30))
    .withTimestampAssigner((e, ts) -> e.eventTimeMillis)
    .withIdleness(Duration.ofMinutes(1));      // ← handle idle partitions
```

| Strategy | Watermark | Use |
|---|---|---|
| **Monotonic** | `max_event_time` | Events are already ordered (rare) |
| **Bounded out-of-orderness** | `max_event_time − delay` | ✅ The standard choice |
| **Percentile-based** | Adaptive from the observed lateness distribution | Variable, unpredictable sources |
| **Punctuated** | A special marker event says "this batch is complete" | When the source can signal completeness |

### Choosing the delay

$$\text{delay} \approx p99 \text{ of observed lateness}$$

| Delay | Completeness | Latency |
|---|---|---|
| 1 s | ~90% | ✅ Fastest |
| 30 s | ~99% | Good balance |
| 5 min | ~99.9% | Slow results |
| 1 h | ~99.99% | Batch-like |

🎯 **Measure your actual lateness distribution before choosing.** Emit `processing_time − event_time` as a histogram; the p99 is your delay. Guessing produces either wrong results or needlessly slow ones.

---

## Late events: three choices

Once the watermark has passed a window's end, an arriving event is **late**. You have exactly three options:

### 1. Drop
```java
// default: late events are silently discarded
```
✅ Simple, bounded state. ❌ **Silently wrong.** Always emit a metric for dropped events — undetected loss is far worse than known loss.

### 2. Allowed lateness — keep the window open longer
```java
.window(TumblingEventTimeWindows.of(Time.minutes(5)))
.allowedLateness(Time.minutes(10))     // keep state 10 min past the watermark
```
The window emits at the watermark, then **emits an updated result** for each late event. Downstream must handle updates (upsert, or retract-and-restate).

⚠️ **Cost:** window state is retained for `lateness` beyond the watermark, multiplying state.

### 3. Side output — handle late data separately
```java
OutputTag<Event> lateTag = new OutputTag<>("late"){};
.sideOutputLateData(lateTag)
// → route to a separate sink, a dead-letter topic, or a nightly batch correction
```

🎯 **The mature production pattern combines all three:** a modest `allowedLateness` for the common case, a **side output** for anything later, and a **nightly batch job** that recomputes from the raw log and corrects the serving store. That combination gives low latency *and* eventual exactness — and proposing it unprompted is a strong senior signal.

---

## The idle partition problem

☠️ **A watermark is the minimum across all input partitions.** If one Kafka partition receives no events (a quiet region, an inactive shard), its watermark never advances — so the **global watermark never advances**, and **no window ever fires**, anywhere.

This is one of the most common and most confusing streaming failures: the job is healthy, throughput is normal, and no output appears at all.

**Fixes:**
```java
WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(30))
    .withIdleness(Duration.ofMinutes(1));   // mark idle sources as "not blocking"
```
Or: ensure every partition receives periodic heartbeat events.

⚠️ **The same problem occurs after a restart or during a backfill** when partitions are consumed at different speeds — one lagging partition holds the whole job's watermark back.

---

## Watermarks and reprocessing

**Replaying history** is where event time proves its worth — and where processing time fails completely.

```
Replaying a week of data in 10 minutes:
  Event time:      windows fire correctly by the events' own timestamps
                   → the output is IDENTICAL to the original run ✅
  Processing time: everything lands in one 10-minute window
                   → completely wrong ❌
```

🎯 **This is the decisive argument for event time**, and the cleanest way to state it: *"with event time, reprocessing produces bit-identical results; with processing time it produces nonsense. That alone makes event time non-negotiable for anything auditable."*

⚠️ During a fast replay, watermarks advance far faster than wall-clock — which is correct, and it's why the delay must be expressed in **event time**, not as a wall-clock timer.

---

## Time-bounded joins

Stream–stream joins need a time bound, or state grows forever.

```sql
SELECT i.ad_id, i.user_id, c.click_time
FROM impressions i
JOIN clicks c
  ON i.impression_id = c.impression_id
 AND c.click_time BETWEEN i.impression_time AND i.impression_time + INTERVAL '30' MINUTE
```

State is retained for the join window plus the watermark delay, then discarded. **Without the time bound, the join buffers every impression forever.**

⚠️ A click 31 minutes after the impression is **not joined** — that's a deliberate business decision encoded in the window, and it should be stated as such, not discovered later.

---

## Debugging watermarks

| Symptom | Likely cause |
|---|---|
| **No output at all** | ☠️ An idle partition holding the watermark back |
| Output is late by exactly the delay | Working as configured — reduce the delay if the latency matters |
| Many events dropped as late | The delay is too small; measure the real distribution |
| Watermark far behind wall-clock | Consumer lag, or a source producing old timestamps |
| Results differ between runs | Processing time is being used somewhere |
| Windows fire immediately with wrong data | Wrong timestamp field, or wrong units (seconds vs milliseconds) |

**The metrics to emit:**
```
currentWatermark                            # per operator
watermark_lag = now() - currentWatermark    # ← the headline health metric
event_lateness_histogram                    # processing_time - event_time
late_events_dropped_total
window_fired_total
```

☠️ **A units mismatch (seconds vs milliseconds) is a surprisingly common bug**: timestamps 1,000× too small put every event in 1970, so the watermark jumps to "now", and **every real event is late and dropped**. The symptom is an empty output with no errors.

---

## The interview answer

> "I'd use event time, because it makes results deterministic — replaying the same data produces identical output, which is what makes backfills and audits possible. Processing time would give different answers on every run.
>
> Watermarks let the system decide when a window is complete. I'd set the delay from the measured p99 of `processing_time − event_time` — say 30 seconds — which trades a small amount of latency for about 99% completeness.
>
> For the remaining 1%, I'd allow a few minutes of lateness so windows can emit corrections, route anything later to a side output, and run a nightly batch job that recomputes from the raw log and corrects the serving store. That gives low latency now and exactness eventually.
>
> The failure mode I'd watch for is idle partitions: the watermark is the minimum across all inputs, so one silent partition stalls the entire job — no output at all, with everything looking healthy. I'd configure idleness detection and alert on watermark lag."

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| Distinguish the three times and say why event time wins | ☐ |
| Define a watermark in one sentence | ☐ |
| Name four generation strategies and the standard one | ☐ |
| Choose a delay from a lateness distribution | ☐ |
| Give the three options for late data and the combined production pattern | ☐ |
| Explain the idle partition problem and its fix | ☐ |
| Give the reprocessing argument for event time | ☐ |
| Explain why stream joins need a time bound | ☐ |
| List the watermark metrics and the units-mismatch bug | ☐ |
| Deliver the interview answer | ☐ |

---

**← Previous** [6.3.2 Windowing Patterns](02-windowing-patterns.md)
**Next →** [6.3.4 Delivery Guarantees](04-delivery-guarantees.md)
