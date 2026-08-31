# 6.6.2 — Time-Series Patterns

> **Part 6 · Big Data · Real-time & Analytics · Chapter 2 of 3**
> Metrics, IoT, and events: enormous write volume, time-ordered, and mostly queried in ranges.

---

## 🧒 ELI5 — Explain Like I'm 5

You're writing down the temperature **every second, forever**.

That's a very particular kind of list, and it has unusual properties:

- **It only ever grows at the end.** You never go back and change what the temperature was at 3pm last Tuesday. *(Append-only.)*
- **Every entry has a time**, and the times are in order. *(Naturally sorted.)*
- **You almost always ask "what happened between X and Y?"**, never "find me all the 21-degree readings ever." *(Range queries.)*
- **Yesterday matters a lot; last year barely at all.** *(Recency bias.)*
- **Neighbouring numbers are nearly identical** — 20.1, 20.1, 20.2, 20.2. *(Which means they squash down beautifully.)*

Because the data is so predictable, you can do things you couldn't with normal data:

- **Squash it hard.** Instead of "20.1, 20.1, 20.2", write "20.1, then +0, then +0.1." Tiny.
- **Throw away detail as it ages.** Keep every second for a day, every minute for a month, every hour for a year. Nobody needs per-second data from 2019.
- **Delete a whole day at once** by throwing away that day's page, instead of hunting for individual entries.

A time-series database is a database that assumes all of this — and is dramatically better at it as a result.

---

## The workload

| Property | Consequence |
|---|---|
| **Append-only, time-ordered** | ✅ LSM storage; no read-modify-write |
| **Extremely high write rate** | Millions of points/sec |
| **Range queries dominate** | Partition and sort by time |
| **Recent data is hot** | Tiered storage: recent in memory/SSD, old in object storage |
| **Values are highly correlated** | ✅ 10–100× compression |
| **Old data is deleted wholesale** | Drop partitions, don't `DELETE` rows |
| **Aggregation over exact rows** | Downsampling and rollups are acceptable |

**The data model:**
```
measurement + tags (indexed labels) + timestamp → value(s)

cpu_usage{host="web-1", region="eu", core="0"}  1756633451  87.2
```

| Element | Role |
|---|---|
| **Metric / measurement** | What is being measured |
| **Tags / labels** | Dimensions to filter and group by — **indexed** |
| **Timestamp** | The primary sort key |
| **Value(s)** | Usually a float |

---

## Cardinality: the defining constraint

$$\text{cardinality} = \prod_i |\text{tag}_i|$$

Every unique combination of tag values is a separate **series**, each with its own index entry and its own storage stream.

```
cpu_usage{host, region, core}
  1,000 hosts × 5 regions × 32 cores = 160,000 series      ✅ fine

http_requests{host, endpoint, status, user_id}
  1,000 × 200 × 10 × 10,000,000 = 2 × 10¹³ series          ☠️ catastrophic
```

☠️ **Putting a high-cardinality value in a tag is the single most destructive mistake in time-series systems**, and it is astonishingly common. Adding `user_id`, `request_id`, `session_id`, `email`, or a URL with parameters as a label creates a new series **per value**, and the index — which must be held in memory — explodes. Prometheus will OOM. InfluxDB will crawl. The failure is sudden and total.

| Never a tag | Use instead |
|---|---|
| `user_id`, `session_id`, `trace_id` | Logs or traces, not metrics |
| Full URL with query parameters | A normalised route template (`/users/:id`) |
| Email, IP address | Aggregate, or use logs |
| Error message text | A small enum of error classes |
| Timestamps | The timestamp field itself |

🎯 **The rule to state:** *"metrics are for aggregate, low-cardinality dimensions; anything with per-request identity belongs in logs or traces, not in labels."* This is the single most useful piece of practical advice about metrics systems, and interviewers who have operated one will recognise it immediately.

**Detecting a cardinality problem before it's fatal:**
```promql
topk(10, count by (__name__)({__name__=~".+"}))       # series count per metric
prometheus_tsdb_head_series                            # total active series
```

---

## Compression

Time-series data compresses far better than general data, because of Gorilla-style encoding (from Facebook's 2015 paper):

| Technique | Applies to | Effect |
|---|---|---|
| **Delta-of-delta** | Timestamps | Regular intervals compress to ~1 bit per point |
| **XOR** | Float values | Similar consecutive floats share most bits |
| **Run-length** | Repeated values | Long constant runs collapse |
| **Dictionary** | Tag values | Repeated labels stored once |

🔢 **Typical result: 16 bytes per raw point (8-byte timestamp + 8-byte value) compresses to ~1.4 bytes** — a ~10× reduction, and much more for slowly-changing values. This is why a time-series database stores a year of per-second metrics where a relational database would need terabytes.

---

## Retention and downsampling

**The core pattern:** keep full resolution briefly, then progressively coarser.

| Age | Resolution | Retention | Relative size |
|---|---|---|---|
| 0–24 h | 1 s | 1 day | 86,400 points |
| 1–7 days | 1 min | 7 days | 10,080 points |
| 7–90 days | 5 min | 90 days | 25,920 points |
| 90 days – 2 y | 1 h | 2 years | 17,520 points |

🔢 **Total: ~140,000 points instead of 63 million** for two years at 1-second resolution — a **450× reduction**, while keeping full detail exactly where it's needed (recent debugging).

```sql
-- TimescaleDB
SELECT add_retention_policy('metrics', INTERVAL '90 days');

CREATE MATERIALIZED VIEW metrics_hourly
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', ts) AS bucket, host,
       avg(value), max(value), min(value)
FROM metrics GROUP BY bucket, host;

SELECT add_continuous_aggregate_policy('metrics_hourly',
  start_offset => INTERVAL '3 hours', end_offset => INTERVAL '1 hour',
  schedule_interval => INTERVAL '1 hour');
```

⚠️ **Downsample to `avg`, `min`, `max`, `count`, and `sum` — not just the average.** An average alone hides spikes: a one-second 100% CPU spike inside an hour of 5% usage averages to ~5%, and the incident becomes invisible in the historical data. Keeping `max` preserves it.

☠️ **Percentiles cannot be averaged.** You cannot compute a daily p99 from 24 hourly p99s. To downsample percentiles you must store a **mergeable sketch** (t-digest, DDSketch), not the percentile value. This is one of the most common analytical errors in monitoring, and stating it is a strong signal.

---

## The systems

| System | Model | Best for |
|---|---|---|
| **Prometheus** | Pull-based, local TSDB, PromQL | ✅ Infrastructure monitoring; the de-facto standard |
| **VictoriaMetrics / Thanos / Mimir** | Prometheus-compatible, long-term, clustered | Multi-cluster, long retention |
| **InfluxDB** | Push-based, TSM (LSM-like) | IoT, application metrics |
| **TimescaleDB** | ✅ Postgres extension | When you want SQL, joins, and one database |
| **ClickHouse** | Columnar OLAP | Very high volume; analytics plus time-series |
| **Apache Druid / Pinot** | Real-time OLAP | Sub-second queries over high-cardinality event data |
| **Graphite / Whisper** | Legacy fixed-size files | Older estates |
| **Managed** (Datadog, CloudWatch, Grafana Cloud) | Hosted | When you'd rather not operate it |

⚖️ **TimescaleDB is the pragmatic choice more often than people expect:** it's Postgres, so you get SQL, joins to relational data, transactions, and one system to operate — with hypertables giving automatic time partitioning, compression, and continuous aggregates. **Reach for a dedicated TSDB when write volume genuinely exceeds what Postgres can take**, not by default.

### Prometheus's pull model

```yaml
scrape_configs:
  - job_name: 'api'
    scrape_interval: 15s
    static_configs: [{ targets: ['api-1:9090', 'api-2:9090'] }]
```

| ✅ Pull advantages | ❌ Costs |
|---|---|
| Prometheus knows if a target is **down** (a failed scrape is a signal) | Doesn't fit short-lived jobs → needs a Pushgateway |
| No need to configure a destination in every service | Requires network reachability to every target |
| Scrape interval controlled centrally | Firewalls and NAT complicate it |
| Easy to run a second Prometheus for testing | |

🎯 **"A failed scrape is itself a signal" is the underrated advantage.** With push-based metrics, a service that dies simply stops sending — indistinguishable from a service with nothing to report. Pull makes absence detectable.

---

## Query patterns

```promql
rate(http_requests_total[5m])                          # per-second rate from a counter
histogram_quantile(0.99, rate(latency_bucket[5m]))     # p99 from a histogram
sum by (region) (rate(http_requests_total[5m]))        # aggregate away dimensions
predict_linear(disk_free[6h], 4*3600) < 0              # will the disk fill in 4h?
```

| Metric type | Meaning | Query with |
|---|---|---|
| **Counter** | Monotonically increasing | ✅ `rate()` — never the raw value |
| **Gauge** | Goes up and down | Raw value, `avg`, `max` |
| **Histogram** | Bucketed distribution | `histogram_quantile()` |
| **Summary** | Client-computed quantiles | ❌ **Cannot be aggregated across instances** |

☠️ **Two classic errors, both worth knowing:**
1. **Graphing a counter's raw value** — it only ever rises, so the graph is a meaningless ramp. Counters must be wrapped in `rate()` or `increase()`.
2. **Averaging percentiles across instances** — `avg(p99_by_instance)` is not the fleet p99 and can be wildly wrong. Use **histograms**, which are mergeable; that's exactly why Prometheus histograms exist and why summaries are discouraged for aggregation.

---

## Design patterns for time-series storage

| Pattern | Detail |
|---|---|
| **Partition by time** | ✅ Enables pruning *and* instant deletion by dropping partitions |
| **Sort within partitions by (series, time)** | Range scans become sequential reads |
| **Wide rows / column families** | Cassandra: one partition per `(series, day)`, clustered by timestamp |
| **Bucket the partition key** | ☠️ Prevents unbounded partitions — see below |
| **Pre-aggregate on write** | Maintain rollups as data arrives |
| **Separate hot and cold** | Recent in memory/SSD, historical in object storage |

```sql
-- Cassandra: bucketing is mandatory, not optional
CREATE TABLE metrics (
  series_id text,
  day date,                        -- ← the bucket
  ts timestamp,
  value double,
  PRIMARY KEY ((series_id, day), ts)
) WITH CLUSTERING ORDER BY (ts DESC);
```

⚠️ **Without the `day` bucket, one series' partition grows forever** — and Cassandra partitions have a practical ceiling around 100 MB. This is the [unbounded partition problem](../../05-scaling-data-storage/02-data-partitioning/02-advanced-partitioning.md#composite-and-hierarchical-keys) in its most common form.

---

## Sizing

```
Hosts                 10,000
Metrics per host      500
Scrape interval       15 s
Series                10,000 × 500 = 5,000,000
Points per second     5,000,000 / 15 ≈ 333,000/s
Raw bytes/point       16 B → ~1.4 B compressed
Storage per day       333,000 × 86,400 × 1.4 B ≈ 40 GB/day
15-day retention      ≈ 600 GB
With rollups to 2 y   + ~50 GB
```

🎯 **Series count, not point count, is what determines memory.** Prometheus holds an index entry per active series; 5 million series is roughly 15–20 GB of RAM before any query load. **That is why cardinality control is the operational priority.**

---

## ☠️ Failure modes

| Mistake | Consequence |
|---|---|
| **High-cardinality labels** | Index explosion; OOM; the system dies |
| Storing raw data forever | Unbounded cost |
| Downsampling to `avg` only | Spikes vanish from history |
| Averaging percentiles | Wrong numbers presented as authoritative |
| Graphing counters without `rate()` | Meaningless ever-rising lines |
| No partition bucketing | Unbounded partitions |
| `DELETE` instead of dropping partitions | Slow, and it creates tombstones |
| Using a relational database for high-volume metrics | Index bloat; writes fall over |
| No cardinality monitoring | The explosion is discovered by an outage |
| Summaries where histograms are needed | Cannot aggregate across instances |

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| List the seven properties of time-series workloads | ☐ |
| Compute cardinality and identify a dangerous label | ☐ |
| State the metrics-vs-logs rule for identity fields | ☐ |
| Explain delta-of-delta and XOR compression, with the byte figures | ☐ |
| Design a retention and downsampling ladder, with the reduction factor | ☐ |
| Explain why you must keep `max`, not just `avg` | ☐ |
| Explain why percentiles need sketches to downsample | ☐ |
| Compare six time-series systems and justify TimescaleDB | ☐ |
| Explain pull vs push and the failed-scrape signal | ☐ |
| Name the four metric types and their query patterns | ☐ |
| Explain the two classic query errors | ☐ |
| Size a metrics system and identify the memory driver | ☐ |

---

**← Previous** [6.6.1 Materialized Views](01-materialized-views-streaming.md)
**Next →** [6.6.3 Analytics Architecture](03-streaming-analytics-architecture.md)
