# 3.3.12 — Distributed Caching

> **Part 3 · Scaling Services · Caching · Chapter 12 of 18**
> One cache node isn't enough. Spreading data across many, without losing it all when one leaves.

---

## 🧒 ELI5 — Explain Like I'm 5

One fridge isn't big enough for the whole street, so you get **ten fridges** and put them in ten houses.

Now: **which fridge holds the milk?**

The obvious answer: *"count the letters in 'milk', divide by ten, use the remainder."* Everyone does the same sum, so everyone goes to the same fridge. It works!

Until **one fridge breaks and you're down to nine.** Now everybody's sums change — you divide by nine instead of ten — and **almost everything is in the wrong fridge.** Practically the whole street's food has to be re-fetched from the shop at once. The shop collapses.

The clever fix: imagine the ten fridges arranged **around a circular table**, each at a spot decided by its name. To find the milk, you also put "milk" at a spot on the circle, then **walk clockwise until you bump into a fridge**. That's the one.

Now when a fridge breaks, only the items that *pointed at that fridge* have to move — **one tenth of everything, not all of it.** Everyone else's food stays exactly where it was.

That circle is **consistent hashing**, and it's why removing a cache node is survivable.

---

## Why distribute?

| Reason | Detail |
|---|---|
| **Capacity** | The working set exceeds one machine's RAM |
| **Throughput** | Redis is single-threaded for commands — one node tops out around 100–200k ops/s |
| **Availability** | One node's death shouldn't lose 100% of the cache |
| **Network bandwidth** | One node's NIC caps total data served |
| **Blast radius** | Failure affects 1/N of keys, not all of them |

---

## The partitioning problem

### ❌ Modulo hashing

```
node = hash(key) % N
```

Simple, evenly distributed, and **catastrophic on any membership change**.

| Nodes | Keys that move |
|---|---|
| 10 → 9 | ~90% |
| 10 → 11 | ~91% |

☠️ Losing one node out of ten doesn't invalidate 10% of the cache — it invalidates **~90%**, because every key's modulo changes. Your origin then receives near-full traffic with a cold cache. **This is how a single cache node failure takes down a whole system.**

### ✅ Consistent hashing

Map both nodes and keys onto a ring (hash space `0 … 2³²−1`). A key belongs to the first node clockwise from its position.

```
        node-A(120)
       ╱           ╲
key-x(90)          key-y(200)
     │                  │
 node-D(50)         node-B(250)
       ╲           ╱
        node-C(400)
```

**Adding or removing a node only affects keys between it and its predecessor** — roughly `1/N` of the keyspace.

| Nodes | Keys that move on removal |
|---|---|
| 10 → 9 | ~10% |
| 100 → 99 | ~1% |

⚠️ **Virtual nodes are required.** With one ring position per physical node, distribution is badly uneven (some nodes get 3× their share) and removing a node dumps its entire load onto exactly one neighbour. Give each physical node **100–500 virtual positions**; load then evens out and a departing node's keys spread across all survivors.

Full treatment, with code: [Consistent Hashing](../../05-scaling-data-storage/02-data-partitioning/03-consistent-hashing.md).

---

## Client-side vs server-side sharding

| | **Client-side** (Memcached, Redis Cluster clients) | **Proxy** (Twemproxy, Envoy, mcrouter) | **Server-side** (Redis Cluster) |
|---|---|---|---|
| Who routes | The client library | A proxy layer | The nodes redirect (`MOVED`/`ASK`) |
| Extra hop | None | Yes (~0.3 ms) | None (after the client learns the map) |
| Client complexity | High | Low | Medium (cluster-aware client) |
| Rebalancing | Client config change | Proxy config | Automatic slot migration |
| Multi-key ops | Client must group by node | Proxy handles some | Same-slot only |

**Redis Cluster** uses **16,384 hash slots** rather than a continuous ring: `slot = CRC16(key) mod 16384`, and slots are assigned to nodes. Moving a node means reassigning slots — a coarser, more manageable unit than raw ring positions.

```
CLUSTER NODES        # who owns which slots
CLUSTER KEYSLOT foo  # which slot a key maps to
```

---

## Multi-key operations across shards

☠️ `MGET a b c` fails in Redis Cluster if the keys live in different slots — as do transactions, Lua scripts touching multiple keys, and `SUNION`.

**Hash tags** force related keys into the same slot:

```
user:{44}:profile      ┐
user:{44}:settings     ├── only the text inside {} is hashed
user:{44}:preferences  ┘   → all three share one slot
```

⚠️ **Hash tags reintroduce hot spots.** If `{tenant:big-customer}` tags thousands of keys, they all land on one node. Use tags only for genuinely small, genuinely co-accessed groups.

**Cluster-aware clients** (redis-py, Lettuce, go-redis) automatically split an `MGET` by slot and issue parallel commands — so batching still works, it just becomes N parallel round trips instead of one. Know that this is happening; it changes your latency model.

---

## Memcached vs Redis for a distributed cache

| | Memcached | Redis |
|---|---|---|
| Data types | Strings only | Strings, hashes, lists, sets, sorted sets, streams, bitmaps, HLL |
| Threading | ✅ Multi-threaded — scales with cores | Single-threaded for commands (I/O threads in 6+) |
| Memory efficiency | ✅ Slab allocator, lower per-key overhead | Higher overhead, richer structures |
| Clustering | Client-side only | Redis Cluster, or client-side |
| Persistence | None | RDB + AOF |
| Replication | None | Yes |
| Eviction | LRU (segmented) | Multiple policies |
| Atomic operations | CAS, incr/decr | Rich, plus Lua scripting |
| Ops complexity | ✅ Very simple | More features, more to operate |

⚖️ **Choose Memcached** for a pure, large, simple string cache where you want maximum memory efficiency and multi-core throughput. **Choose Redis** for everything else — counters, rate limits, sorted sets, pub/sub, locks — which in practice is most systems, because you end up wanting those structures anyway.

---

## Replication within the cache tier

```
Shard 1: primary-1  →  replica-1a, replica-1b
Shard 2: primary-2  →  replica-2a
Shard 3: primary-3  →  replica-3a
```

| Purpose | Benefit |
|---|---|
| **Failover** | A replica is promoted; that shard's data survives |
| **Read scaling** | Reads served from replicas (with `READONLY` in Redis Cluster) |
| **Blast radius** | Losing a primary costs a brief failover, not a cold shard |

⚠️ **Redis replication is asynchronous.** A failover can lose the last few writes. For a cache that's acceptable — it's a cache. For a rate-limit counter, a distributed lock, or a session store you've treated as durable, it is **not**, and you should say so if a design depends on it.

Details: [Cache High Availability](13-cache-high-availability.md).

---

## Data placement strategies

| Strategy | How | Use |
|---|---|---|
| **Hash sharding** | `hash(key)` → node | ✅ Default; even distribution |
| **Range sharding** | Key ranges per node | Rarely useful in caches; hot ranges |
| **Directory** | A lookup service maps key → node | Flexible, but a new critical dependency |
| **Locality-aware** | Key → the node nearest the requester | Multi-region caches |
| **Replicated (every node holds everything)** | Small, extremely hot datasets | Feature flags, config, small reference tables |

**Fully-replicated small caches deserve a mention:** for a 10 MB config or flag dataset read on every request, replicating it to every node (or every application instance) eliminates network hops entirely. Different problem, different answer.

---

## Hot keys in a distributed cache

Sharding spreads *keys*, not *traffic*. One key receiving 30% of requests still lands on one node.

```
Total 100,000 QPS across 10 nodes = 10,000 QPS each ✅
But one celebrity key gets 30,000 QPS → all on ONE node ❌
```

| Mitigation | How |
|---|---|
| **Local cache with a 1 s TTL** | ✅ 10,000 QPS → 1 QPS per instance. Almost always sufficient |
| **Key replication** | Store as `hot:{k}:r0..r9`; readers pick one at random; writers update all |
| **Read from replicas** | Spreads reads across that shard's replicas |
| **CDN** | If the content is public |
| **Client-side coalescing** | Concurrent requests for the same key share one fetch |

**Detection:**
```bash
redis-cli --hotkeys                       # requires maxmemory-policy allkeys-lfu
redis-cli MONITOR | head -10000 | ...     # ⚠️ MONITOR is expensive — brief samples only
```

Better: track top-K keys in the client library and export a metric. Then hot-key mitigation can be **automatic** — promote a key to the local cache once it exceeds a QPS threshold.

---

## Rebalancing

Adding capacity means moving slots/keys while serving traffic.

```
1. Add the new node to the cluster
2. Assign it slots (Redis Cluster: CLUSTER SETSLOT ... MIGRATING / IMPORTING)
3. Migrate keys in batches
4. During migration, the source node answers ASK redirects for moved keys
5. Finalise slot ownership
```

⚠️ **What to watch:**
- **Migration adds load** to both source and destination — do it during off-peak.
- **Big keys stall migration** (`MIGRATE` is blocking per key) — a 1 GB key can freeze a node for seconds.
- **Clients must handle `MOVED`/`ASK`** — cluster-aware clients do; naive ones fail.
- **Hit rate dips** during and just after migration as clients refresh their slot maps.

⚖️ **Or don't rebalance at all.** Because it's a *cache*, you can simply change the ring and accept a temporary hit-rate drop — as long as your origin can absorb the extra misses. **Check that first**: if the origin can't take a 20% miss-rate spike, you must migrate rather than reshard, and you should know that before you need to.

---

## Cross-region caching

| Approach | Consistency | Latency | Notes |
|---|---|---|---|
| **Independent cache per region** | Each region's cache is independent | ✅ Local | ✅ Usually right. Invalidation must be broadcast to all regions |
| **Global replicated cache** | Async replication between regions | Local reads, replicated writes | Conflict and lag complexity |
| **Single global cache** | One region owns it | ❌ 100 ms+ for remote regions | Almost never right |

🎯 **The standard design: independent regional caches, with invalidation events broadcast globally.** Each region misses independently (so the origin sees N× the cold-start load, one per region) but reads are always local. Broadcast invalidation via a global topic (Kafka MirrorMaker, SNS fan-out, or your CDC stream replicated cross-region).

---

## ☠️ Failure modes

| Failure | Consequence |
|---|---|
| Modulo hashing | One node lost → ~90% cache invalidation → origin overload |
| Consistent hashing without virtual nodes | Uneven load; a removal dumps everything on one neighbour |
| Multi-key ops across slots | Runtime errors when you scale from one node to a cluster |
| Overusing hash tags | Artificial hot shards |
| Hot key on one shard | That node saturates while others idle |
| Assuming Redis replication is synchronous | Lost writes on failover — fine for a cache, fatal for a lock |
| Rebalancing at peak | Migration load compounds traffic load |
| Very large keys | Blocking migrations and blocking reads |
| Client not cluster-aware | `MOVED` errors surface as application failures |
| One global cache for a multi-region app | Cross-region latency on every read |

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| Explain why modulo hashing moves ~90% of keys | ☐ |
| Explain consistent hashing and the 1/N property | ☐ |
| Explain why virtual nodes are required | ☐ |
| Compare client-side, proxy, and server-side sharding | ☐ |
| Explain Redis Cluster's 16,384 slots | ☐ |
| Use hash tags correctly and name their hazard | ☐ |
| Choose Memcached vs Redis with a reason | ☐ |
| State the asynchronous-replication caveat | ☐ |
| Give five hot-key mitigations for a sharded cache | ☐ |
| Describe rebalancing and its four hazards | ☐ |
| Design cross-region caching with global invalidation | ☐ |

---

**← Previous** [3.3.11 Eviction & Sizing](11-eviction-and-sizing.md)
**Next →** [3.3.13 Cache High Availability](13-cache-high-availability.md)
