# 6.5.3 — Error Handling

> **Part 6 · Big Data · Pipeline Operations · Chapter 3 of 3**
> Bad data is not an exception; it is a constant. Design for it.

---

## 🧒 ELI5 — Explain Like I'm 5

You're sorting a huge pile of forms. One form is filled in wrong — the date says "banana".

You have four choices:

1. **Stop everything.** Nobody gets any results because of one silly form. *(Fail fast — sometimes right, usually not.)*
2. **Ignore it and carry on.** ✅ Nine hundred thousand forms processed, one skipped. But **if nobody knows it was skipped**, you have quietly wrong numbers and no idea why. *(Skip — but only with a record.)*
3. **Put it in a "weird forms" tray**, carry on, and look at the tray later. ✅ **Nothing is lost, nothing is blocked, and someone can fix it.** *(Dead-letter queue.)*
4. **Guess.** Assume the date is today. ⚠️ Sometimes fine, sometimes it silently corrupts everything downstream. *(Default values — dangerous.)*

Option 3 is almost always right. And the rule that makes it work is: **the weird-forms tray must have someone looking at it.** A tray nobody empties is just a slower way of throwing things away.

---

## Types of failure

| Type | Example | Retry helps? |
|---|---|---|
| **Transient infrastructure** | Network blip, S3 throttling, node preemption | ✅ Yes |
| **Resource exhaustion** | OOM, disk full, quota exceeded | ⚠️ Only after changing something |
| **Bad record** | Malformed JSON, wrong type, missing required field | ❌ No — it will always fail |
| **Schema change** | An upstream column removed or retyped | ❌ No — needs a code change |
| **Logic bug** | Divide by zero on an edge case | ❌ No |
| **Upstream unavailable** | Source API down, empty response | ✅ Yes, with backoff |
| **Silent data quality** | ☠️ Rows arrive but values are wrong | ❌ Only tests catch this |

🎯 **The last row is the dangerous one.** Everything above it produces an error someone can see. Silent quality failures produce a **green pipeline and wrong dashboards** — and are typically discovered weeks later by someone noticing a number looks odd.

---

## Failure granularity

| Level | Response |
|---|---|
| **Record** | Skip to a dead-letter store, continue the batch |
| **Partition / task** | Retry; if it keeps failing, quarantine that partition |
| **Job run** | Retry with backoff; alert after N failures |
| **Pipeline** | Halt dependent jobs; page someone |

⚠️ **Match the response to the level.** Failing an entire 10-hour job because one record in ten million is malformed is a bad trade. Silently skipping 40% of records because a schema changed is a much worse one.

**The threshold rule:**
```python
if bad_records / total_records > 0.001:      # 0.1%
    fail_the_job("bad record rate exceeded threshold")
else:
    write_to_dlq(bad_records)                 # and continue
```

🎯 **A bad-record *rate* threshold is the right mechanism**, and it's worth naming. A handful of bad records is normal; a sudden jump means something structural changed upstream and continuing would produce quietly wrong output.

---

## Dead-letter queues for data pipelines

```python
def process(record):
    try:
        return transform(validate(record))
    except ValidationError as e:
        dlq.write({
            "raw": record,
            "error": str(e),
            "error_type": type(e).__name__,
            "pipeline": PIPELINE_NAME,
            "version": CODE_VERSION,          # ← which build produced this
            "partition": current_partition,
            "occurred_at": now(),
        })
        metrics.increment("records.dlq", tags={"reason": type(e).__name__})
        return None
```

| Requirement | Why |
|---|---|
| **Store the raw record** | You cannot reprocess what you didn't keep |
| **Store the error and the code version** | Diagnosis without reproduction |
| **Store the partition** | So a fix can be targeted |
| **Alert on rate, not on presence** | A trickle is normal; a spike is an incident |
| **Alert on *age*** | ☠️ A DLQ nobody empties is silent data loss |
| **Provide a replay path** | Most DLQ records are fixable; you want them back |

☠️ **The unmonitored DLQ is the most common way data is lost while every dashboard stays green.** The pipeline reports success, the DLQ fills, and nobody looks. **Monitor the oldest unprocessed DLQ record**, not just the count.

---

## Retry policy

| Setting | Guidance |
|---|---|
| **Max attempts** | 3–5 for transient errors; **0** for validation errors |
| **Backoff** | Exponential with **jitter** — see [Retries](../../02-microservices-and-dataflow/01-synchronous-communication/06-retries.md) |
| **Retry only what's retryable** | Classify the exception; never retry a `ValidationError` |
| **Idempotency** | ✅ Mandatory — a retry must not duplicate output |
| **Timeout** | Bound each attempt; a hung task is worse than a failed one |

```python
@retry(
    retry=retry_if_exception_type((ConnectionError, ThrottlingError)),
    stop=stop_after_attempt(5),
    wait=wait_exponential_jitter(initial=1, max=60),
)
def fetch_source_data(date): ...
```

⚠️ **Retrying a non-idempotent write is how one failure becomes duplicated data.** Partition-overwrite writes make retries free ([Backfill](02-backfill-reprocessing.md#2-idempotency)); append-only writes make them dangerous.

---

## Schema evolution failures

The most common recurring cause of pipeline breakage.

| Upstream change | Effect | Mitigation |
|---|---|---|
| New column added | Usually harmless | Select explicit columns; don't `SELECT *` into a typed model |
| Column removed | ❌ Breaks everything referencing it | Schema contract + alerting |
| Type changed (`int` → `string`) | ❌ Cast failures | Explicit casts with error handling |
| Column renamed | ❌ Equivalent to remove + add | Contract |
| Enum value added | Downstream `CASE` falls through to a default | ✅ Always have an explicit `else` that flags unknowns |
| Nullability changed | Nulls where none were expected | Test `not_null` |

**Defences:**

| Defence | Detail |
|---|---|
| **Schema registry with compatibility rules** | `BACKWARD` compatibility so consumers upgrade first |
| **Explicit column selection** | Never `SELECT *` into a downstream model |
| **A schema test in the pipeline** | Fail loudly at ingestion, not silently three models later |
| **Data contracts** | The producing team owns the schema and is alerted when it breaks consumers |
| **A permissive raw layer, strict staging** | Land anything; validate on the way to staging |

🎯 **"Land permissively, validate on read" is the pattern that keeps ELT robust:** the raw layer accepts whatever arrives (so ingestion never breaks and no data is lost), and the staging layer applies strict typing and validation. A schema change then fails in a *recoverable* place, with the raw data safely stored.

---

## Detecting silent failures

The failures that don't raise exceptions.

| Check | Catches |
|---|---|
| **Freshness** — "the newest row is < N hours old" | ☠️ A dead upstream that returns empty successfully |
| **Volume** — "row count within X% of the trailing average" | A partial load; a filter accidentally excluding most rows |
| **Null rate** — "not_null_pct within bounds" | An upstream field that stopped being populated |
| **Distribution** — "mean/p95 within historical bounds" | A unit change (cents → pounds), a broken join |
| **Referential integrity** | Orphaned foreign keys from a partial load |
| **Reconciliation to source** | ✅ Totals match the system of record |
| **Uniqueness** | A duplicate load |

```yaml
- name: fct_orders
  tests:
    - dbt_utils.recency: { datepart: hour, field: created_at, interval: 6 }
    - dbt_expectations.expect_table_row_count_to_be_between:
        min_value: 8000
        max_value: 20000
```

☠️ **The empty-success failure is the classic:** an upstream API returns `200 OK` with `[]` because a token expired. The job succeeds, loads zero rows, the incremental model adds nothing, and every dashboard shows a flat line that looks like a quiet business day. **Only a volume or freshness test catches it.**

🎯 **Reconciliation to the source system is the strongest check available.** "Our warehouse says 8,412 orders yesterday; the production database says 8,412" is a complete end-to-end verification. Run it daily and alert on any discrepancy.

---

## Partial failure and consistency

☠️ A pipeline that writes three tables and fails after the second leaves the warehouse **internally inconsistent** — and downstream models will happily join them and produce wrong results.

| Approach | Detail |
|---|---|
| **Write to staging, then swap atomically** | ✅ All-or-nothing at the visible layer |
| **Transactional multi-table writes** | Where the warehouse supports it (Snowflake, Postgres) |
| **Table format transactions** | ✅ Iceberg/Delta commit atomically |
| **Publish a completion marker** | Downstream waits for `_SUCCESS` before reading |
| **Version the whole dataset** | Consumers read a pinned snapshot version |

⚠️ **The `_SUCCESS` marker convention is simple and effective** (inherited from Hadoop): a job writes its output, then writes an empty `_SUCCESS` file. Consumers refuse to read a partition without one. It costs nothing and prevents an entire class of half-read failures.

---

## Alerting that works

| Alert on | Not on |
|---|---|
| ✅ SLA breach ("data not fresh by 07:00") | Every individual task retry |
| ✅ Bad-record rate above threshold | The presence of any bad record |
| ✅ Volume anomaly | Normal variance |
| ✅ DLQ oldest-record age | DLQ count alone |
| ✅ Reconciliation mismatch | — |
| ✅ Job failure after all retries | The first transient failure |

☠️ **Alert fatigue is the real failure mode.** A pipeline that pages on every transient retry trains everyone to ignore it, and the one genuine alert is missed. **Alert on *outcomes* (data is wrong or late), not on *events* (a task retried).**

**Every alert needs:** what broke, which partition/date, the likely cause, a link to logs, and a link to the runbook. An alert that says only "dag_orders failed" costs the responder ten minutes of orientation before they can start.

---

## Runbooks

```markdown
## Alert: fct_daily_revenue is stale

**Impact:** the executive dashboard shows yesterday's data. Finance uses this at 09:00.

**Diagnose**
1. Check the upstream freshness test: `dbt test -s source:raw.orders`
2. If raw is stale → the ingestion connector is the problem, not the transformation
3. If raw is fresh → check the model run logs

**Common causes**
- Source API token expired → rotate it (see: secrets runbook)
- Volume test failed on a legitimate low-volume day → override with justification
- Upstream schema change → check the schema-registry alert channel

**Fix**
- Rerun: `dbt run -s fct_daily_revenue --full-refresh` (~12 min)
- If the raw partition is bad: backfill that date (see: backfill runbook)

**Escalate** if not resolved within 30 min (finance needs it by 09:00)
```

🎯 **A runbook is worth more than the alert.** Alerting without a runbook just moves the confusion from the dashboard to a person at 3 a.m. Every alert should link to one, and the runbook should be updated every time it turns out to be wrong.

---

## Observability for pipelines

```
pipeline_run_duration_seconds{pipeline, status}
pipeline_records_processed_total{pipeline}
pipeline_records_failed_total{pipeline, reason}
pipeline_dlq_oldest_seconds{pipeline}            # ← the important one
data_freshness_seconds{table}                    # ← and this one
data_row_count{table, partition}
data_quality_test_failures_total{table, test}
pipeline_cost_estimate{pipeline}
```

**Data lineage** turns "this number looks wrong" into "these three upstream models feed it, and this one failed at 04:12." Without it, every data-quality investigation starts from scratch. dbt, Dagster, OpenLineage, and warehouse-native lineage all provide it — and it is one of the highest-value things a data platform can offer.

---

## ☠️ Failure modes

| Mistake | Consequence |
|---|---|
| Failing the whole job on one bad record | Fragile; one malformed row blocks everything |
| Skipping bad records silently | Quietly wrong data, undetectable |
| DLQ with no monitoring | Silent data loss |
| Retrying non-retryable errors | Wasted time; the same failure N times |
| Retrying non-idempotent writes | Duplicated data |
| No freshness or volume tests | Empty-success failures go unnoticed |
| Substituting default values for bad data | Corruption that looks like real data |
| Partial writes visible to consumers | Downstream joins produce wrong results |
| Alerting on every retry | Alert fatigue; real alerts ignored |
| No runbook | Every incident is solved from first principles |
| No lineage | Impact analysis is impossible |

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| Classify seven failure types by retryability | ☐ |
| Match response to failure granularity | ☐ |
| Explain the bad-record rate threshold | ☐ |
| Design a DLQ record and its alerting | ☐ |
| Write a retry policy that only retries retryable errors | ☐ |
| Name six schema-evolution failures and their defences | ☐ |
| Explain "land permissively, validate on read" | ☐ |
| Name six silent-failure checks, especially freshness and volume | ☐ |
| Describe the empty-success failure | ☐ |
| Explain `_SUCCESS` markers and atomic swaps | ☐ |
| Distinguish alerting on outcomes vs events | ☐ |
| Write a runbook with impact, diagnosis, fix, and escalation | ☐ |

---

**← Previous** [6.5.2 Backfill & Reprocessing](02-backfill-reprocessing.md)
**Next →** [6.6.1 Materialized Views](../06-realtime-and-analytics/01-materialized-views-streaming.md)
