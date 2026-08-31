# 4.1.3 — SSTable

> **Part 4 · Data Storage · Data Structures Behind Databases · Chapter 3 of 4**
> Sorted String Table: an immutable, sorted file. The building block of every LSM tree.

---

## 🧒 ELI5 — Explain Like I'm 5

An SSTable is **a page of names, written in alphabetical order, in permanent ink.**

Three rules:

1. **It's sorted.** Names go in A-to-Z order. That means to find "Kettle" you can jump to the middle, see "M", jump back — a few jumps and you're there. You never read the whole page.
2. **It's never changed.** Once written, that page is finished. Forever. If Kettle changes their phone number, you **do not** cross anything out — you write a *new* page with the new number, and that new page wins because it's newer.
3. **It comes with a little summary card** taped to the front: "this page covers Adams to Zhang; Adams is at line 1, Baker at line 100, Chen at line 200..." So you don't even have to search the whole page — the card tells you roughly where to jump.

**Why permanent ink?** Because you can write a whole page from top to bottom in one smooth motion, without ever going back to fix something. That's enormously faster than hunting for the right line and squeezing something in.

**The catch:** now you have lots of pages, and the same name might be on several of them. So when you look someone up, you check the newest page first, then the next, and so on. And every so often you **merge all the pages into one tidy new page** and throw the old ones away.

---

## The format

```
┌──────────────────────────────────────────┐
│ DATA BLOCKS (sorted by key, compressed)  │
│   block 0: apple→1  ant→2  arch→9        │
│   block 1: bat→4    bell→7  bird→3       │
│   block 2: cat→8    cow→5   crow→6       │
├──────────────────────────────────────────┤
│ INDEX BLOCK (sparse)                     │
│   apple → offset 0                       │
│   bat   → offset 4096                    │
│   cat   → offset 8192                    │
├──────────────────────────────────────────┤
│ BLOOM FILTER (may this key be here?)     │
├──────────────────────────────────────────┤
│ METADATA / FOOTER                        │
│   min key, max key, entry count,         │
│   compression codec, checksums, seq range│
└──────────────────────────────────────────┘
```

| Component | Purpose |
|---|---|
| **Data blocks** | Sorted key-value pairs, typically 4–64 KB, compressed individually |
| **Sparse index** | One entry per *block*, not per key — small enough to keep in memory |
| **Bloom filter** | Answers "definitely not present" without any disk I/O |
| **Footer** | Min/max key for range pruning, checksums for corruption detection |

🔢 **Why the index is sparse:** one entry per 64 KB block means a 1 GB SSTable has ~16,000 index entries (a few hundred KB) instead of millions. The index fits in memory; a lookup is then **one** disk read of the right block, followed by a binary search within it.

---

## The three properties that make it work

### 1. Sorted
- Binary search within a block: O(log n) with no I/O.
- **Range scans are sequential reads** — the fastest thing a disk does.
- **Merging two SSTables is a linear merge**, like merge sort. This is what makes compaction affordable.

### 2. Immutable
- **Writing is purely sequential** — no seeks, no read-modify-write.
- **No locking for readers.** An SSTable never changes, so readers need no coordination with writers. This is a major concurrency win.
- Safe to cache aggressively, copy, ship to another node, or back up — it can't change under you.
- Corruption is detectable via checksums and repairable by re-fetching from a replica.

### 3. Self-describing
Min/max keys let a reader **skip an entire file** without opening it. The Bloom filter lets it skip a file it *might* need. Both are essential to keeping read amplification manageable.

---

## Reading a key

```
1. Check the Bloom filter        → "definitely not here" ⇒ skip the file entirely (no I/O)
2. Check min/max in the footer   → out of range ⇒ skip
3. Binary search the sparse index → find the block offset
4. Read that one block (one disk read)
5. Decompress, binary search within the block
6. Return the value, or continue to the next (older) SSTable
```

🎯 **Steps 1 and 2 are the point.** Without them, a read would touch every SSTable at every level. With them, a typical read touches **one** file's data block.

### Bloom filter arithmetic

$$\text{false positive rate} \approx \left(1 - e^{-kn/m}\right)^k$$

🔢 At **10 bits per key** with 7 hash functions, the false-positive rate is about **1%**. So 99% of "key not in this file" questions are answered from memory with zero I/O, for 10 bits per key — a 1-million-key SSTable needs ~1.25 MB of filter.

⚠️ **A Bloom filter has no false negatives.** "Not present" is always true; "possibly present" may be wrong, costing one wasted block read. This asymmetry is exactly what you need here.

⚠️ **Bloom filters only help point lookups.** A range scan must open every SSTable whose min/max overlaps the range — which is why range queries are LSM's weaker case.

---

## Writing an SSTable

```python
def flush_memtable(memtable, path):
    with open(path, "wb") as f:
        index, bloom = [], BloomFilter(len(memtable), fp_rate=0.01)
        block = []
        for key, value in memtable.sorted_items():        # already sorted in memory
            bloom.add(key)
            block.append((key, value))
            if block_size(block) >= 64 * 1024:
                index.append((block[0][0], f.tell()))     # first key → offset
                f.write(compress(serialize(block)))
                block = []
        if block:
            index.append((block[0][0], f.tell()))
            f.write(compress(serialize(block)))
        index_offset = f.tell();  f.write(serialize(index))
        bloom_offset = f.tell();  f.write(bloom.serialize())
        f.write(footer(index_offset, bloom_offset,
                       min_key=..., max_key=..., checksum=...))
```

Note: **one pass, no seeks, no reads.** The MemTable was already sorted, so writing is a pure sequential stream. This is why LSM flushes are so fast.

---

## Deletes: tombstones

You cannot modify an immutable file, so a delete is **a write**:

```
SSTable 3 (newest):  key=alice  value=<TOMBSTONE>
SSTable 2:           key=alice  value={name: "Alice B"}
SSTable 1 (oldest):  key=alice  value={name: "Alice A"}

Read alice → newest wins → tombstone → return "not found"
```

⚠️ **Tombstone consequences:**
- Deleted data still occupies space until compaction removes it.
- A range scan over many tombstones is slow — you read entries only to discard them.
- Tombstones **cannot be dropped immediately** in a distributed system: if a replica was offline during the delete and the tombstone were removed too early, that replica would resurrect the deleted row during repair. Hence Cassandra's `gc_grace_seconds` (default 10 days).

☠️ **Cassandra's classic operational failure: "tombstone hell."** A queue-like table where rows are written and deleted repeatedly accumulates millions of tombstones; a range query then reads them all and times out. **Cassandra is not a queue** — that anti-pattern is the reason.

---

## Compaction

Multiple SSTables must be merged, or reads touch too many files and space is wasted.

```
Before:  [a→1, c→2, e→3]  [a→9, b→5]  [c→7, d→8]     3 files, 'a' and 'c' duplicated
After:   [a→9, b→5, c→7, d→8, e→3]                    1 file, newest values only
```

**A k-way merge of sorted files** — linear in total size, sequential I/O throughout.

| Strategy | Behaviour | Trade-off |
|---|---|---|
| **Size-tiered (STCS)** | Merge SSTables of similar size | ✅ Low write amplification; ❌ high space amplification (up to 2×) and read amplification |
| **Levelled (LCS)** | Levels of increasing size, non-overlapping within a level | ✅ Low read and space amplification; ❌ high write amplification (~10–30×) |
| **Time-window (TWCS)** | Group by time window | ✅ Ideal for time-series with TTL — whole files expire together |
| **Universal/hybrid** | Tunable middle ground | RocksDB configurability |

⚖️ **The choice is a direct RUM trade** ([Chapter 1](01-data-structures-behind-databases.md#the-rum-conjecture)): STCS optimises writes, LCS optimises reads and space, TWCS optimises time-series deletion.

Detailed treatment: [LSM Tree](04-lsm-tree.md).

---

## SSTables in the wild

| System | Usage |
|---|---|
| **Bigtable** | The original design (2006 paper) |
| **LevelDB / RocksDB** | The embedded engine under many databases |
| **Cassandra / ScyllaDB** | SSTables per table, per node |
| **HBase** | HFiles — the same idea |
| **InfluxDB** | TSM files — SSTables specialised for time-series |
| **Kafka** | Segment files are conceptually similar: immutable, sequential, indexed |
| **Parquet / ORC** | Immutable columnar files with footers and statistics — same philosophy, columnar layout |
| **Lucene segments** | Immutable inverted-index segments, merged in the background |

🎯 **The pattern generalises:** *immutable, sorted, self-describing files merged in the background* is one of the most reused ideas in data infrastructure. Recognising Parquet and Lucene segments as the same idea in different clothing is a strong signal.

---

## Optimisations worth knowing

| Optimisation | Effect |
|---|---|
| **Block compression** (Snappy, LZ4, zstd) | 2–10× smaller; less I/O, more CPU. LZ4 for hot data, zstd for cold |
| **Prefix compression** | Sorted keys share prefixes (`user:1001`, `user:1002`) — store the delta only |
| **Block cache** | Cache decompressed blocks in memory — the LSM equivalent of a buffer pool |
| **Partitioned index** | For very large files, index the index |
| **Ribbon / cuckoo filters** | Better space/accuracy than Bloom filters |
| **Per-level Bloom tuning** | More bits at lower (larger, colder) levels |
| **Checksums per block** | Detect corruption early; repair from a replica |

---

## SSTable vs B-tree page: the essential contrast

| | B-tree page | SSTable |
|---|---|---|
| Mutability | ✅ Modified in place | ❌ Immutable |
| Size | Fixed (4–16 KB) | Whole file (MBs to GBs) |
| Writes | Random | ✅ Sequential |
| Reader locking | Latches required | ✅ None needed |
| Space overhead | Fragmentation, ~50% min occupancy | ✅ Fully packed |
| Compression | Per page, limited | ✅ Per block, excellent (sorted data compresses well) |
| Finding a key | Traverse the tree | Bloom → index → one block |
| Reclaiming space | Vacuum / rebuild | Compaction |

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| Draw the SSTable layout with all four components | ☐ |
| Explain why the index is sparse and what it costs | ☐ |
| State the three properties and what each enables | ☐ |
| Walk the six-step read path | ☐ |
| Quote Bloom filter bits-per-key and false-positive rate | ☐ |
| Explain why Bloom filters don't help range scans | ☐ |
| Explain tombstones and `gc_grace_seconds` | ☐ |
| Explain tombstone hell and why Cassandra isn't a queue | ☐ |
| Compare STCS, LCS, and TWCS as RUM trades | ☐ |
| Recognise Parquet and Lucene segments as the same pattern | ☐ |

---

**← Previous** [4.1.2 B-tree](02-b-tree.md)
**Next →** [4.1.4 LSM Tree](04-lsm-tree.md)
