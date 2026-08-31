# 6.5.2 — Backfill & Reprocessing

> **Part 6 · Big Data · Pipeline Operations · Chapter 2 of 3**
> Recomputing history without breaking production. The operation that separates designed pipelines from accidental ones.

---

## 🧒 ELI5 — Explain Like I'm 5

You've been keeping a tally of sales every day for a year. Today you discover that for the **whole year**, you've been forgetting to include Saturdays.

Now you have to **redo the last 365 days** — while still counting today's sales, and without confusing anyone looking at the numbers in the meantime.

Three things make this survivable or awful:

1. **Did you keep the receipts?** If you threw them away and only kept the daily totals, you **cannot** redo it. Ever. *(Keep raw data.)*
2. **If you redo the same day twice by accident, does it double?** If your method is "add today's sales to the total", redoing a day counts it twice. If it's "**replace** that day's total with the correct number", redoing is harmless. *(Idempotency — and this is the whole trick.)*
3. **Can you redo it quietly?** If you overwrite the real numbers while people are looking at them, they see nonsense mid-repair. Better: **write the corrected version somewhere else, check it, then swap.** *(Shadow and swap.)*

Get those three right and a backfill is boring. Get any one wrong and it's an incident.

---

## Why you'll need it

| Reason | Frequency |
|---|---|
| A bug in the transformation logic | Common |
| A new column or metric needed for history | Common |
| A late-arriving correction from a source system | Common |
| Business logic changed (e.g. a new revenue-recognition rule) | Occasional |
| A pipeline failed and left a gap | Occasional |
| Onboarding a new data source with history | At launch |
| A schema change requiring re-derivation | Occasional |
| Recovering from data loss or corruption | Rare, urgent |

🎯 **Backfills are routine, not exceptional.** A pipeline that cannot be backfilled safely is a pipeline that cannot be *fixed* — every bug becomes permanent. **Design for it on day one**, because retrofitting idempotency into a running pipeline is far harder than building it in.

---

## The three prerequisites

### 1. Raw data retention

You cannot recompute what you did not keep.

```
✅ Raw events in S3/Iceberg, indefinitely, partitioned by date
❌ Only the aggregated output
```

⚠️ **This is the single decision that makes backfills possible or impossible**, and it is made long before you need it. Cheap object storage means there is rarely a good reason not to keep raw data. (Exception: data you are legally required to delete.)

### 2. Idempotency

> **Running the same job twice must produce the same result as running it once.**

| Pattern | Idempotent? |
|---|---|
| `INSERT INTO totals SELECT ...` | ❌ No — duplicates on rerun |
| `DELETE WHERE date = X; INSERT ...` | ⚠️ Yes, but not atomic — a crash between them loses data |
| ✅ `INSERT OVERWRITE PARTITION (date=X)` | ✅ **Yes, atomically** |
| ✅ `MERGE ... ON key WHEN MATCHED UPDATE WHEN NOT MATCHED INSERT` | ✅ Yes |
| `UPDATE counter SET n = n + 1` | ❌ No |
| `UPDATE counter SET n = <computed total>` | ✅ Yes |

🎯 **Partition-level overwrite is the canonical answer**, and it's worth stating as a rule: *"every job writes whole partitions, so rerunning a partition replaces it rather than appending to it. That makes backfill just 'rerun these partitions', with no special code path."*

### 3. Parameterised time ranges

```python
# ❌ The job assumes "now"
def run():
    process(data_for(date.today()))

# ✅ The job takes the date it is processing
def run(execution_date):
    process(data_for(execution_date))
```

⚠️ **Any use of `now()` inside transformation logic breaks backfills**, because reprocessing last March will use today's date. This includes: `CURRENT_DATE` filters, `now() - interval '1 day'` windows, and "is this record recent?" flags. **Every temporal reference must come from the execution parameter**, not the wall clock.

---

## Backfill strategies

### Strategy 1 — Partition-by-partition rerun

```bash
for d in $(seq_dates 2026-01-01 2026-08-31); do
  run_job --date=$d --parallelism=high
done
```

| ✅ | ❌ |
|---|---|
| Simple; reuses the normal code path | Slow if serial |
| Resumable — you know exactly which partitions are done | Can overwhelm shared resources |
| Naturally idempotent with partition overwrite | |

**Run partitions in parallel** where they're independent (usually they are), but **throttle** — a backfill running at full speed will starve the live pipeline and hammer downstream systems.

### Strategy 2 — Shadow table and swap

```
1. Build the corrected output into a NEW table (`fct_revenue__backfill`)
2. Verify it against the current table for overlapping periods
3. Atomically swap: rename, or repoint a view/alias
4. Keep the old table for a rollback window, then drop it
```

✅ **Zero user-visible inconsistency**, and rollback is instant. **This is the right pattern whenever the output is being actively read.**

### Strategy 3 — Dual-write and cut over

For streaming pipelines: run v2 alongside v1 into a separate view, let it catch up, verify, then switch readers. This is [Kappa reprocessing](../04-hybrid-architectures/02-kappa-architecture.md#reprocessing-the-operational-procedure).

### Strategy 4 — Lambda-style correction

Keep the live path running and apply a **correction pass** that recomputes and overwrites specific partitions. Least disruptive; good for correcting late-arriving data routinely rather than as an incident.

---

## The operational concerns

| Concern | Handling |
|---|---|
| **Resource contention** | A separate compute pool or queue; run off-peak; cap parallelism |
| **Downstream load** | Throttle writes; a backfill can easily DDoS the serving store |
| **Cost** | 🔢 Reprocessing a year of data costs roughly a year of daily compute, compressed into hours — budget for it |
| **Duration** | Estimate before starting: `(partitions × per-partition time) / parallelism` |
| **Ordering** | Some models must be backfilled in dependency order |
| **Partial failure** | Track per-partition status so you can resume, not restart |
| **Communication** | Tell consumers that historical numbers will change |

☠️ **The unthrottled backfill is a classic self-inflicted outage:** a backfill job runs at maximum parallelism, saturates the warehouse or the serving database, and takes down the live pipeline and the dashboards it was meant to fix. **Always cap backfill concurrency below live capacity.**

**A progress table makes backfills resumable:**
```sql
CREATE TABLE backfill_progress (
  job_name text, partition_date date, status text,      -- pending|running|done|failed
  attempts int DEFAULT 0, error text, updated_at timestamptz,
  PRIMARY KEY (job_name, partition_date)
);
```
Workers claim `pending` partitions atomically, mark `done` on success. A crash resumes exactly where it stopped, and you can see progress and failures at a glance.

---

## Verification: never swap blind

| Check | Method |
|---|---|
| **Row counts** | Per partition, old vs new |
| **Aggregate totals** | Sums and distinct counts should match where logic is unchanged |
| **Spot checks** | Sample specific rows and inspect manually |
| **Expected delta** | ✅ If the fix should change ~2% of rows, verify it changed ~2%, not 0% or 90% |
| **Downstream impact** | Check dependent models and dashboards |
| **Reconcile to source** | Totals should match the source system |

🎯 **The "expected delta" check is the most valuable and the most often skipped.** A backfill that changes *nothing* means the fix didn't apply. A backfill that changes *everything* means you broke something else. **Predict the magnitude of the change before you run it**, then verify against your prediction.

---

## Streaming backfills

Harder, because the pipeline is running while you reprocess.

| Approach | Notes |
|---|---|
| **Replay from the log** | Only if retention covers the period |
| **Replay from the archive** | ✅ Read Iceberg/S3 as a bounded source with the same code ([Unified Processing](../04-hybrid-architectures/03-batch-stream-unification.md)) |
| **Separate backfill job** | Batch job corrects historical partitions; the stream keeps handling live data |
| **Bootstrap a new state store** | Rebuild state from history, then attach to the live stream |

☠️ **Never replay a pipeline with external side effects.** Reprocessing a notification pipeline would resend every notification. **Structure pipelines so state derivation is separate from side effects**, and only replay the former.

⚠️ **Event time makes streaming backfills correct.** Replaying a year at 100× speed produces identical windows because they're keyed to event time. With processing time, everything lands in one window and the output is meaningless.

---

## Late-arriving data: the routine case

Not every correction is an incident. Some data is simply late by design.

| Pattern | Handling |
|---|---|
| **Lookback window** | Each run reprocesses the last N days, not just yesterday |
| **Late-arrival partition** | Route very late records to a separate partition, merged periodically |
| **Scheduled re-runs** | Reprocess yesterday, 7 days ago, and 30 days ago on every run |

```sql
{% if is_incremental() %}
  WHERE event_date >= (SELECT MAX(event_date) - INTERVAL '3 days' FROM {{ this }})
{% endif %}
```

🎯 **A lookback window turns "backfill" from an incident into a routine.** Most late data arrives within a few days; reprocessing a rolling window absorbs it automatically. Choose the window from the measured lateness distribution — the same p99 reasoning as [watermarks](../03-stream-processing/03-event-time-watermarks.md).

⚠️ **The lookback only works if writes are idempotent** — otherwise reprocessing three days triples three days of data.

---

## Communicating a backfill

Historical numbers changing is a **user-facing event**, even when the change is a fix.

```
□ Announce before: what will change, by roughly how much, and when
□ Explain why: "we were excluding Saturday transactions"
□ Record it: a changelog entry with the date range and the reason
□ Announce after: confirm completion and the actual magnitude of change
□ Update any external reports already published from the old numbers
```

☠️ **Silently changing last quarter's revenue number destroys trust in the data platform** — more thoroughly than the original bug did. Someone presented that number to a board. **Communication is part of the operation, not an optional extra**, and saying so is a maturity signal.

---

## Designing for backfill: the checklist

```
□ Raw data retained, immutably, partitioned by time
□ All jobs parameterised by execution date — no now() in transformation logic
□ Writes are idempotent (partition overwrite or merge on a key)
□ Partitioning matches the natural backfill unit (usually day)
□ Jobs can run out of order and in parallel across partitions
□ A progress table makes runs resumable
□ Backfill compute is isolated or throttled
□ A verification step compares old and new, including expected delta
□ Outputs can be built in a shadow table and swapped atomically
□ Lineage is documented so downstream impact is knowable
□ A lookback window handles routine lateness automatically
□ Side-effecting steps are separated from state derivation
```

🎯 **In an interview:** *"I'd make every job idempotent by writing whole partitions with insert-overwrite and parameterising by execution date, so a backfill is just 'rerun these partitions' with no special code. Raw data stays in Iceberg indefinitely so anything can be recomputed. Backfills run in a throttled pool, build into a shadow table, and get verified against an expected delta before an atomic swap — and I'd announce the change to consumers, because historical numbers moving is a user-facing event."*

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| Give eight reasons backfills happen | ☐ |
| State the three prerequisites | ☐ |
| Explain why `now()` in logic breaks backfills | ☐ |
| Compare four backfill strategies | ☐ |
| Explain the shadow-and-swap pattern | ☐ |
| Describe a resumable progress table | ☐ |
| Explain the expected-delta verification | ☐ |
| Explain why side-effecting pipelines can't be replayed | ☐ |
| Explain lookback windows and their dependence on idempotency | ☐ |
| Explain why communication is part of the operation | ☐ |
| Recite the design-for-backfill checklist | ☐ |

---

**← Previous** [6.5.1 ETL vs ELT](01-etl-vs-elt.md)
**Next →** [6.5.3 Error Handling](03-pipeline-error-handling.md)
