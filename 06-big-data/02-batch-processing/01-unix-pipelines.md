# 6.2.1 — Unix Pipelines

> **Part 6 · Big Data · Batch Processing · Chapter 1 of 4**
> The original data processing system. Every idea in Spark and Flink is here, at small scale.

---

## 🧒 ELI5 — Explain Like I'm 5

Imagine a **line of people passing buckets** to put out a fire.

Each person does exactly **one simple job** and passes the result to the next:

- Person 1: scoop water.
- Person 2: take out the leaves.
- Person 3: pour it on the fire.

Nobody needs to know the whole plan. Each just takes what's handed to them, does their one thing, and passes it on. And here's the clever part: **person 3 starts pouring before person 1 has finished scooping.** Everyone works at once. Nobody waits for the whole lake to be scooped first.

That's a Unix pipeline. Each little program does **one thing well**, and `|` hands the output of one straight into the next, **as it's produced**, not after it's all finished.

Why does this matter for "big data"? Because **it's the same idea, just bigger**: many small steps, each doing one thing, data flowing through, nobody holding the whole lake in their hands. Spark, Flink, and MapReduce are this picture with a thousand people and several buckets each.

---

## The canonical example

```bash
cat access.log |
  awk '{print $7}' |          # extract the URL field
  sort |                       # group identical URLs together
  uniq -c |                    # count each group
  sort -rn |                   # order by count, descending
  head -5                      # top 5
```

**This is a complete MapReduce job**, expressed in five commands:

| Pipeline stage | MapReduce equivalent |
|---|---|
| `awk '{print $7}'` | **Map** — extract the key |
| `sort` | **Shuffle** — group by key |
| `uniq -c` | **Reduce** — aggregate per key |
| `sort -rn \| head` | A second job — global top-N |

🎯 **This equivalence is genuinely worth stating in an interview.** MapReduce did not invent a new paradigm; it distributed one that had existed since 1973. Recognising that makes the distributed version far less mysterious.

---

## The design principles

The Unix philosophy, and why each principle matters at scale:

| Principle | Consequence |
|---|---|
| **Do one thing well** | Small, testable, composable, reusable units |
| **Uniform interface (text streams)** | Any tool composes with any other |
| **Everything is a stream** | Constant memory regardless of input size |
| **No hidden state** | Rerun anything; deterministic and testable |
| **Compose, don't build monoliths** | New pipelines from existing parts, instantly |

⚠️ **The uniform interface is the load-bearing idea.** Because every tool reads and writes lines of text, `n` tools give you `n²` possible combinations. That is exactly why Spark's DataFrame API and SQL succeeded — one uniform data interface that every operation understands.

---

## Streaming and constant memory

```bash
# A 100 GB file, processed in a few MB of RAM
cat huge.log | grep ERROR | awk '{print $3}' | sort -u | wc -l
```

Each stage reads a line, processes it, and passes it on. **Memory stays constant.**

⚠️ **Except `sort`.** `sort` must see all input before producing its first output — it's a **blocking** operator. GNU `sort` handles files larger than RAM by an **external merge sort**: sort chunks in memory, spill them to disk, then merge.

🎯 **This is exactly what a distributed shuffle does**, and the parallel is worth drawing: map tasks produce sorted spill files, and the reduce phase merges them. Spark's shuffle is `sort`'s external merge, spread across machines.

**Blocking vs streaming operators** is a distinction that runs through the whole of Part 6:

| Streaming (pipelined) | Blocking (needs all input) |
|---|---|
| `grep`, `awk`, `sed`, `cut`, `tr`, `head` | `sort`, `uniq` (needs sorted input), `wc -l`, `tac` |
| Constant memory | Memory or disk proportional to input |
| Output starts immediately | Output starts only at the end |

**In Spark:** `map`/`filter` are streaming (narrow) transformations; `groupBy`/`join`/`sort` are blocking (wide) transformations that require a shuffle. **Same distinction, same consequences.**

---

## Parallelism

```bash
# Use all cores for the sort
sort --parallel=8 -S 2G big.txt

# Split work across processes
cat urls.txt | xargs -P 8 -n 100 ./process_batch.sh

# GNU parallel: split a file across N workers, then combine
parallel --pipe --block 100M -j 8 'awk "{s+=\$3} END {print s}"' < huge.csv |
  awk '{s+=$1} END {print s}'
```

That last one is a **complete map-reduce**: split into blocks, compute a partial sum per block in parallel, then combine the partials. The only thing MapReduce adds is doing it **across machines** with fault tolerance.

⚖️ **This matters practically:** a modern laptop with 8 cores and an NVMe drive processes tens of GB per minute. 🔢 **Datasets under ~100 GB usually do not need a cluster** — a single machine with `sort`, `awk`, DuckDB, or Polars will beat a Spark cluster once you count cluster startup time. Saying this is a strong signal of judgement, not of ignorance.

---

## The tools worth knowing

| Tool | Use |
|---|---|
| `grep` / `rg` | Filter lines |
| `awk` | Field extraction, arithmetic, per-key aggregation |
| `sed` | Stream editing |
| `cut` / `paste` | Column operations |
| `sort` / `uniq` | Group and count |
| `join` | ✅ Relational join on **sorted** input |
| `comm` | Set operations on sorted files |
| `tr` | Character translation |
| `xargs` / `parallel` | Parallel execution |
| `jq` | JSON processing |
| `csvkit` / `miller` (`mlr`) | Structured CSV/JSON with awk-like ergonomics |
| `pv` | Progress and throughput monitoring |
| `duckdb` | ✅ **SQL over CSV/Parquet files, in one process** |

### awk does more than people expect

```bash
# Sum a column
awk -F, '{sum += $3} END {print sum}' sales.csv

# Group-by in one pass — this is a hash aggregation
awk -F, '{sum[$2] += $3} END {for (k in sum) print k, sum[k]}' sales.csv

# Filter, transform, and format
awk -F, '$3 > 100 {printf "%s: %.2f\n", $1, $3 * 1.2}' sales.csv
```

🎯 That group-by is a **hash aggregation** — the same algorithm Spark uses for `groupBy` when the key space fits in memory. When it doesn't, Spark spills to disk and merges, exactly as `sort` does.

### DuckDB: the modern answer

```bash
duckdb -c "SELECT url, count(*) c FROM read_csv('access.log')
           GROUP BY url ORDER BY c DESC LIMIT 5"

duckdb -c "SELECT * FROM 's3://bucket/events/*.parquet' WHERE date > '2026-08-01'"
```

⚖️ **DuckDB is the pragmatic modern replacement for a large fraction of "we need Spark" situations:** columnar, vectorised, parallel, reads Parquet directly from object storage, and runs in a single process with no cluster. For datasets up to a few hundred GB it is often faster than a small Spark cluster and vastly simpler to operate.

---

## What Unix pipelines can't do

| Limitation | Why | The distributed answer |
|---|---|---|
| **One machine** | Bounded by local CPU, RAM, disk | Distribute across a cluster |
| **No fault tolerance** | A crash loses everything; rerun from scratch | Task retries, lineage, checkpoints |
| **Text is untyped** | Delimiters break on embedded commas/newlines | Typed formats: Parquet, Avro |
| **No schema** | Column positions are implicit and fragile | Schema registry, typed DataFrames |
| **Awkward joins** | `join` requires pre-sorted input on both sides | Hash joins, broadcast joins, sort-merge joins |
| **No incremental processing** | Rerun the whole thing every time | Partitioned tables, incremental models |
| **Local files only** | | Distributed filesystems, object storage |

☠️ **The text-format fragility is the practical killer.** A CSV with a comma inside a quoted field breaks `cut -d,`. Columnar binary formats (Parquet) with embedded schemas exist precisely because "everything is text" stops working once data has real types and real edge cases.

---

## The lineage to modern systems

| Unix concept | Modern equivalent |
|---|---|
| `\|` | DAG edge / dataflow |
| `grep`, `awk` | `filter`, `map`, `select` |
| `sort` | Shuffle |
| `uniq -c` | `groupBy().count()` |
| `join` | Sort-merge join |
| `xargs -P` | Task parallelism |
| Stdin/stdout | Uniform data interface (DataFrame, Dataset) |
| Exit codes | Task success/failure |
| `make` | Airflow / Dagster DAGs |
| Immutable input files | Immutable partitions; idempotent jobs |

🎯 **The immutability point is the deepest one.** Unix tools never modify their input — they read a file and write a new one. That's why you can rerun any pipeline safely. **Modern data engineering rediscovered this as "idempotent jobs writing immutable partitions"**, and it's the foundation of safe backfills ([Backfill & Reprocessing](../05-pipeline-operations/02-backfill-reprocessing.md)).

---

## Practical lessons that carry over

| Lesson | Application at scale |
|---|---|
| **Filter early** | Push predicates down before shuffles and joins |
| **Project early** | Select only the columns you need — huge in columnar formats |
| **Prefer streaming operators** | Avoid unnecessary sorts and shuffles |
| **Compose small, testable steps** | Each stage independently verifiable |
| **Immutable inputs** | Reruns are always safe |
| **Sample first** | `head -1000` before running on 100 GB; the same discipline saves hours on a cluster |
| **Measure** | `pv` for throughput; the Spark UI for stage timings |

```bash
# The discipline, in one line: develop on a sample, then run on everything
head -10000 huge.log | ./my_pipeline.sh    # iterate in seconds
cat huge.log | pv | ./my_pipeline.sh       # then run for real, with progress
```

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| Write the top-5-URLs pipeline and map it to MapReduce stages | ☐ |
| State the five Unix design principles and why the uniform interface matters | ☐ |
| Distinguish streaming and blocking operators, in Unix and in Spark | ☐ |
| Explain external merge sort and its relation to shuffle | ☐ |
| Write a parallel map-reduce with GNU parallel | ☐ |
| State the ~100 GB single-machine threshold | ☐ |
| Write an `awk` hash aggregation | ☐ |
| Name DuckDB as a cluster alternative | ☐ |
| List seven limitations and their distributed answers | ☐ |
| Explain why immutable inputs make reruns safe | ☐ |

---

**← Previous** [6.1.1 Batch & Stream Overview](../01-overview/01-batch-and-stream-overview.md)
**Next →** [6.2.2 The MapReduce Model](02-mapreduce-model.md)
