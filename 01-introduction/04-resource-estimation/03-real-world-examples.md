# 1.4.3 — Resource Estimation: Real World Examples

> **Part 1 · Introduction · Resource Estimation · Chapter 3 of 3**
> Six complete worked estimates. Read one, then do the next yourself before reading it.

---

## 🧒 ELI5 — Explain Like I'm 5

You've learned how to guess party sizes. Now we do it for six different parties, out loud, so the guessing becomes automatic.

Each one follows the same recipe:

> *how many people → how often do they do the thing → how big is each thing → multiply → round → **and what does that mean?***

The last bit is always the point. *"That's a lot of pizza"* is useless. *"That's more pizza than my oven fits, so we're ordering delivery"* is a decision.

Read the first example. Then **cover the next one and try it yourself** before looking. That's how this chapter works.

---

## Example 1 — URL Shortener

```
ASSUMPTIONS
  New URLs           100 million/month
  Read:write ratio   100:1
  URL length         ~100 B long, 7 B short code
  Row size           ~500 B (code, long_url, owner, timestamps, flags)
  Retention          5 years

WRITES
  100M/month ÷ 30 = 3.3M/day → 33 QPS avg → ~100 QPS peak

READS
  33 QPS × 100 = 3,300 QPS avg → ~10,000 QPS peak

STORAGE
  3.3M/day × 500 B = 1.65 GB/day
  × 365 × 5 years  = ~3 TB
  × 3 (RF)         = ~9 TB

CACHE
  80/20: 20% of links get 80% of reads
  Daily distinct reads ~10M; hot set ~2M links × 500 B = 1 GB
  Cache everything hot in a few GB of Redis.

KEY SPACE
  base62, 7 chars = 62^7 ≈ 3.5 × 10^12 codes
  At 100M/month that lasts ~3,000 years. 7 characters is enough.
```

**So what:**
1. 3 TB total, point lookups by key → **key-value store**, or Postgres with a hash index. Sharding is not needed for size, but may be for the 10k read QPS.
2. Read-heavy 100:1 → **cache-first**; a 95% hit rate leaves 500 QPS on the database, trivially served.
3. 7-character base62 codes suffice — no need for longer keys.
4. Writes at 100 QPS are nothing. **The entire design is a read-path design.**
5. Analytics (click counts) must NOT be a `COUNT(*)` — append clicks to a stream and aggregate.

---

## Example 2 — Twitter / X

```
ASSUMPTIONS
  DAU                200M
  Tweets per user    0.5/day
  Timeline views     20/day per user
  Tweet size         300 B text + 200 B metadata = 500 B
  Media              10% of tweets, 1 MB avg
  Avg followers      200 (median ~50; heavy tail to 100M+)

WRITES
  Tweets/day  200M × 0.5 = 100M  → 1,200 QPS avg → ~3,500 peak
  Media/day   10M × 1 MB = 10 TB/day

READS
  Timeline views/day  200M × 20 = 4 × 10^9 → 46,000 QPS → ~140,000 peak
  Each view = ~20 tweets → ~2.8M tweet-reads/sec

FAN-OUT (push model)
  100M tweets/day × 200 avg followers = 2 × 10^10 timeline writes/day
                                       = 230,000 writes/sec average
                                       ≈ 700,000/sec at peak

STORAGE
  Tweets:    100M × 500 B = 50 GB/day → 18 TB/year → 55 TB with RF=3
  Media:     10 TB/day → 3.6 PB/year
  Timelines: 200M users × 800 tweet IDs × 8 B = 1.3 TB (cache, not disk)
```

**So what:**
1. **Fan-out on write is 200× more work than fan-out on read** (230k/s vs 1.2k/s) — but it moves the cost off the read path, where the 140k QPS lives. That is the correct trade for a read-heavy system.
2. **Celebrities break it.** One tweet from a 100M-follower account = 100M timeline writes. Hybrid: push for normal accounts, pull-and-merge at read time for the few thousand accounts above a follower threshold. See [Push vs Pull in Twitter Timeline](../../03-scaling-services/04-dataflow/03-push-vs-pull-twitter-timeline.md).
3. Timelines are 1.3 TB of pure IDs → **keep them in Redis**, capped at ~800 entries per user, and rebuild on demand for inactive users.
4. Media at 3.6 PB/year → object storage + CDN. Never in the database.
5. Tweet text at 18 TB/year → a sharded KV or wide-column store keyed by `tweet_id`.

---

## Example 3 — YouTube

```
ASSUMPTIONS
  DAU              2 billion
  Videos watched   5/day per user
  Uploads          500 hours of video per minute
  Avg watch        10 minutes
  Bitrate served   average 2 Mbps (mixed devices/qualities)
  Renditions       5 (144p to 4K) ≈ 1.6× original size stored

READS
  Views/day   2B × 5 = 10^10 → 115,000 QPS metadata → ~350,000 peak
  Concurrent streams (peak): assume 5% of DAU watching = 100M concurrent
  Egress = 100M × 2 Mbps = 200 Tbps

UPLOADS
  500 h/min = 30,000 h/day... (note: 500 h/min × 60 × 24 = 720,000 h/day)
  720,000 h/day × 1 GB/h (source, compressed) = 720 TB/day
  Stored with renditions: × 1.6 = ~1.15 PB/day
  Per year: ~420 PB/year

TRANSCODING
  720,000 h/day of source × 5 renditions = 3.6M rendition-hours/day
  If 1 CPU-hour transcodes 0.5 h of video → 7.2M CPU-hours/day
                                          = 300,000 cores running constantly
```

**So what:**
1. **200 Tbps egress is not servable from origin.** The design is fundamentally a CDN design — origin serves the CDN, the CDN serves the world. Google runs its own edge PoPs precisely for this.
2. **420 PB/year** → object storage with tiering: hot content on SSD-backed edge caches, long tail on cheap cold storage. Popularity is extremely skewed — a tiny fraction of videos gets most views.
3. **300,000 cores of transcoding** → this is the single biggest compute cost. Segment-parallel transcoding, spot/preemptible instances, hardware encoders, and lazy transcoding (only generate 4K if someone actually requests it).
4. Metadata at 350k QPS but tiny in size → cache-first, sharded KV.
5. Uploads must go **direct to blob storage** via presigned URLs; 720 TB/day cannot flow through API servers.

---

## Example 4 — WhatsApp / Chat

```
ASSUMPTIONS
  DAU              2 billion
  Messages         40/day per user sent
  Message size     ~200 B (mostly text) + 100 B metadata
  Media            5% of messages, 500 KB avg
  Group size       avg 5 recipients per message (mix of 1:1 and groups)
  Retention        30 days server-side (delivered messages deleted)

WRITES
  Messages/day  2B × 40 = 8 × 10^10 = 80 billion/day
  → 800,000 QPS average → ~2,000,000 QPS peak
  Fan-out to recipients: × 5 = 4 million deliveries/sec peak

STORAGE (undelivered queue + 30-day retention)
  Text: 80B × 300 B = 24 TB/day → 720 TB for 30 days → 2.2 PB with RF=3
  Media: 4B msgs × 500 KB = 2 PB/day → object storage, 30-day TTL

CONNECTIONS
  2B users, ~50% online at peak = 1 billion concurrent connections
  At ~100,000 connections per well-tuned server → 10,000 connection servers
  Memory: 1B × 10 KB per connection state = 10 TB of RAM across the fleet
```

**So what:**
1. **Read:write ≈ 1:1** — this is a **write-heavy** system, unlike every other example. Caching and replicas do not help. You need write-optimised storage (LSM-based: Cassandra/ScyllaDB) and heavy sharding by `user_id` or `conversation_id`.
2. **1 billion concurrent connections** is the defining constraint. Dedicated connection-holding servers (Erlang/BEAM historically, or Go/epoll), separate from business logic, with a routing layer that maps `user_id → connection server`.
3. Messages are **transient**: delete after delivery. Storage is a queue, not an archive — this is what keeps 2.2 PB manageable instead of exabytes.
4. Media never goes through the message path — upload to blob, send a reference.
5. Ordering matters per conversation → partition by `conversation_id` to get per-conversation ordering for free.

---

## Example 5 — Uber / ride hailing

```
ASSUMPTIONS
  Active drivers (peak)  1,000,000 concurrent
  Location ping          every 4 s
  Rides                  20M/day
  Matching radius query  ~2 km
  Ride record            ~2 KB

LOCATION WRITES
  1M drivers ÷ 4 s = 250,000 writes/sec
  Payload 100 B → 25 MB/s ingest
  Per day: 21.6 billion pings × 100 B = 2.2 TB/day (if retained)

RIDE REQUESTS
  20M/day → 230 QPS avg → ~1,000 QPS peak
  Each triggers a geospatial query over nearby drivers

STORAGE
  Rides: 20M × 2 KB = 40 GB/day → 15 TB/year (×3 = 45 TB)
  Location history: 2.2 TB/day → downsample aggressively or 800 TB/year
```

**So what:**
1. **250k writes/sec of data worth nothing in 10 seconds.** It must not touch the transactional database. Keep current positions in an **in-memory geospatial index** (Redis `GEOADD`, or geohash/S2/H3 cells in a sharded in-memory service).
2. Location *history* is a separate, lossy, downsampled pipeline → stream to Kafka, aggregate, store compacted trajectories only.
3. Ride requests are only ~1,000 QPS — **tiny**. The hard part is not request volume, it's the **matching computation** and the write firehose.
4. Geospatial sharding by cell (S2/H3) gives locality: a 2 km query touches a handful of cells on a few shards.
5. Rides are transactional with state machines and money → relational store, strong consistency, exactly the opposite requirement from the location stream. **Two very different stores in one system**, which is the whole lesson.

---

## Example 6 — Ad click aggregation / analytics

```
ASSUMPTIONS
  Ad impressions   10 billion/day
  Click rate       1% → 100M clicks/day
  Event size       ~500 B (ad_id, user, ts, context, geo)
  Query pattern    dashboards over last 1 h / 1 d / 30 d, grouped by ad and geo
  Freshness        under 1 minute

INGEST
  Impressions: 10^10/day = 115,000 events/sec → ~350,000 peak
  Clicks:      100M/day  = 1,200/sec

RAW STORAGE
  10^10 × 500 B = 5 TB/day → 1.8 PB/year raw
  Compressed columnar (~10:1) → ~180 TB/year

AGGREGATED STORAGE
  Pre-aggregate by (ad_id, minute, country):
  100,000 ads × 1,440 min × 200 countries = 2.9 × 10^10 rows/day worst case
  Realistically sparse: ~10^8 rows/day × 50 B = 5 GB/day → 1.8 TB/year
```

**So what:**
1. **350k events/sec** → a log (Kafka) is the ingestion buffer, not a database. Partition by `ad_id` for aggregation locality.
2. Raw at 1.8 PB/year → keep 7 days hot in a columnar store, archive the rest to object storage as Parquet.
3. Dashboards must never scan raw events → **pre-aggregate with a stream processor** (Flink) into per-minute rollups; that's 1,000× less data.
4. **Exactly-once matters** — double-counted clicks mean incorrect billing. Requires idempotent aggregation keyed by `event_id`, or Flink's checkpointed transactional sinks. See [Delivery Guarantees](../../06-big-data/03-stream-processing/04-delivery-guarantees.md).
5. Late events (mobile devices offline) → **watermarks and allowed lateness**, plus a nightly batch reconciliation for correctness. This is the classic Lambda/Kappa decision.

---

## The pattern across all six

| System | Dominant constraint | Consequence |
|---|---|---|
| URL shortener | Read QPS | Cache-first, tiny data |
| Twitter | Fan-out volume | Precompute timelines; hybrid for celebrities |
| YouTube | Egress bandwidth + storage | CDN is the architecture; blob + tiering |
| WhatsApp | Write rate + concurrent connections | Write-optimised store; dedicated connection tier |
| Uber | Ephemeral write firehose + geo queries | In-memory geo index, separate from transactional store |
| Ad analytics | Event rate + query aggregation | Log → stream aggregation → columnar rollups |

🎯 **The meta-lesson:** every system has **one dominant constraint**, and the architecture is mostly a response to that one number. Your job in the estimation phase is to find it and name it. *"The defining number here is 200 Tbps of egress — everything else follows from that"* is the sentence you're aiming for.

---

## Practice set (do these before reading any solution)

| System | Find the dominant constraint |
|---|---|
| Google Drive / Dropbox (500M users, 10 GB each) | Storage, dedup, sync bandwidth |
| Netflix (250M subscribers, 2 h/day) | Egress and encoding storage |
| Ticketmaster (a 100k-seat venue, on-sale moment) | Instantaneous concurrency and strong consistency |
| Slack (20M DAU, 100 messages/day, search over all history) | Search index size and per-workspace isolation |
| A metrics system (1M hosts × 500 metrics × every 10 s) | Cardinality and write rate (50M points/sec) |
| Stripe-like payments (10k TPS) | Correctness/idempotency, not volume |

For each: assumptions → QPS → storage → bandwidth → **so what**. Three minutes each, out loud, timed.

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| Complete any of the six estimates from a blank page in 3 min | ☐ |
| Identify the dominant constraint before designing | ☐ |
| Recognise read-heavy vs write-heavy from the ratio | ☐ |
| Know when data must be ephemeral rather than stored | ☐ |
| Explain why fan-out on write is worth 200× the work | ☐ |
| Convert every estimate into an architectural decision | ☐ |

---

**← Previous** [1.4.2 QPS and System Design](02-qps-and-system-design.md)
**Next →** [1.5.1 Microservices and Monolithic Architecture](../05-microservices/01-microservices-vs-monolithic.md)
