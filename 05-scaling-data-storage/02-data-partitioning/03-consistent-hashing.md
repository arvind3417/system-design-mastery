# 5.2.3 — Consistent Hashing

> **Part 5 · Scaling Data Storage · Data Partitioning · Chapter 3 of 4**
> One idea, twenty lines of code, and it's the reason distributed caches and databases can add nodes without falling over.

---

## 🧒 ELI5 — Explain Like I'm 5

You have ten boxes and lots of toys. **Which box does each toy go in?**

**The obvious way:** count the letters in the toy's name, divide by ten, use the remainder. Everyone does the same sum, so everyone finds the toy. Works perfectly.

**Until you get an eleventh box.** Now everyone divides by *eleven* instead of ten — and **almost every toy is suddenly in the wrong box.** You'd have to take out and re-sort nearly everything. For *one* new box.

**The clever way:** draw a **big circle** on the floor. Put the ten boxes around the edge at spots decided by their names. To find a toy's box: work out the toy's spot on the circle, then **walk clockwise until you bump into a box.** That's its box.

Now add an eleventh box. It sits somewhere on the circle. Only the toys **between the new box and the box before it** need to move — about a tenth of them. **Everything else stays exactly where it is.**

One extra trick: if you only give each box *one* spot on the circle, some boxes end up with a huge slice and others with a sliver. So you give each box **a hundred spots**, scattered around. Now the slices even out beautifully.

---

## The problem it solves

```
Modulo hashing:  node = hash(key) % N

N: 10 → 11    ≈ 91% of keys remap
N: 10 → 9     ≈ 90% of keys remap
```

☠️ **Losing one cache node out of ten invalidates ~90% of your cache**, not 10%. The origin then receives near-full traffic with a cold cache — the [avalanche](../../03-scaling-services/03-caching/15-failure-modes.md) that takes systems down.

**Consistent hashing:** adding or removing a node remaps only ~**1/N** of keys.

$$\text{keys moved} = \frac{K}{N}$$

| Change | Modulo | Consistent hashing |
|---|---|---|
| 10 → 11 nodes | ~91% | ✅ ~9% |
| 100 → 101 | ~99% | ✅ ~1% |
| 10 → 9 | ~90% | ✅ ~10% |

---

## How it works

**1. Map everything onto a ring** — a hash space of `0 … 2³²−1`, wrapped into a circle.

**2. Place nodes** at `hash(node_id)`.

**3. Place keys** at `hash(key)`.

**4. A key belongs to the first node clockwise from its position.**

```
                    0 / 2³²
                       │
        node-D(50) ────┼──── node-A(120)
              ╱                    ╲
      key-x(30)                  key-y(200)
            │                        │
      node-C(3800M)              node-B(2500M)
              ╲                    ╱
               ────────────────────

  key-x(30)  → walk clockwise → node-D(50)
  key-y(200) → walk clockwise → node-B(2500M)
```

**5. Removing a node:** its keys go to the next node clockwise. **Nothing else moves.**
**6. Adding a node:** it takes over the keys between itself and its predecessor. **Nothing else moves.**

---

## Virtual nodes — not optional

With one ring position per physical node:

| Problem | Detail |
|---|---|
| **Uneven distribution** | Random placement gives some nodes 3× their fair share |
| **Uneven failover** | A departing node dumps *all* its keys onto **one** neighbour, which may then be overloaded and fail too — a cascade |
| **No heterogeneity** | A machine with twice the RAM can't take twice the load |

**Fix: give each physical node V virtual positions** (typically 100–500).

```
node-A → "node-A#0", "node-A#1", ... "node-A#149"   → 150 ring positions
```

| Virtual nodes per physical node | Standard deviation of load |
|---|---|
| 1 | ~100% (wildly uneven) |
| 10 | ~30% |
| 100 | ✅ ~10% |
| 500 | ~5% |

🔢 **100–200 virtual nodes gives roughly 10% load variance**, which is acceptable, and it means a departing node's keys are redistributed across **all** survivors rather than one. **Weighted nodes** fall out naturally: give a bigger machine 300 virtual nodes instead of 150.

---

## Implementation

```python
import bisect, hashlib

class ConsistentHashRing:
    def __init__(self, nodes=None, vnodes=150):
        self.vnodes = vnodes
        self.ring = {}          # hash → node
        self.sorted_keys = []   # sorted hashes, for binary search
        for n in (nodes or []):
            self.add_node(n)

    @staticmethod
    def _hash(key: str) -> int:
        return int(hashlib.blake2b(key.encode(), digest_size=8).hexdigest(), 16)

    def add_node(self, node, weight=1):
        for i in range(self.vnodes * weight):
            h = self._hash(f"{node}#{i}")
            self.ring[h] = node
            bisect.insort(self.sorted_keys, h)

    def remove_node(self, node, weight=1):
        for i in range(self.vnodes * weight):
            h = self._hash(f"{node}#{i}")
            del self.ring[h]
            self.sorted_keys.remove(h)          # O(n); fine for infrequent changes

    def get_node(self, key):
        if not self.ring:
            return None
        h = self._hash(key)
        idx = bisect.bisect_right(self.sorted_keys, h) % len(self.sorted_keys)
        return self.ring[self.sorted_keys[idx]]

    def get_nodes(self, key, count):
        """The first `count` DISTINCT nodes clockwise — for replication."""
        if not self.ring:
            return []
        h = self._hash(key)
        idx = bisect.bisect_right(self.sorted_keys, h) % len(self.sorted_keys)
        out = []
        for i in range(len(self.sorted_keys)):
            node = self.ring[self.sorted_keys[(idx + i) % len(self.sorted_keys)]]
            if node not in out:
                out.append(node)
                if len(out) == count:
                    break
        return out
```

| Operation | Complexity |
|---|---|
| Lookup | O(log(N × V)) — a binary search |
| Add node | O(V log(N × V)) |
| Remove node | O(V × N) with a list; O(V log) with a proper structure |
| Memory | O(N × V) — 100 nodes × 150 vnodes = 15,000 entries. Trivial |

🎯 **`get_nodes` is the important extra.** Replication takes the next `R` **distinct** nodes clockwise — which is exactly how Cassandra and Dynamo choose replicas. The "distinct" part matters: without it, several virtual nodes of the same physical node would count as separate replicas, and you'd have R copies on one machine.

---

## Rack and zone awareness

Replicas on three nodes in the same rack all die with the rack. Real implementations walk the ring and **skip nodes in a failure domain already used**:

```python
def get_nodes_zone_aware(self, key, count, zone_of):
    picked, zones = [], set()
    for node in self._walk(key):
        z = zone_of(node)
        if node in picked or z in zones:
            continue
        picked.append(node); zones.add(z)
        if len(picked) == count:
            return picked
    return picked
```

Cassandra's `NetworkTopologyStrategy` does exactly this. **Mentioning rack awareness shows you've thought about correlated failure**, which is the whole point of replication.

---

## Where it's used

| System | Usage |
|---|---|
| **Amazon Dynamo / DynamoDB** | The original published design (2007) |
| **Cassandra / ScyllaDB** | Token ring; virtual nodes (`num_tokens`) |
| **Riak** | Ring with virtual nodes |
| **Memcached clients** | Client-side consistent hashing (ketama) |
| **Redis Cluster** | ⚠️ **Not** consistent hashing — 16,384 fixed hash slots (equivalent goals, different mechanism) |
| **Envoy / NGINX / HAProxy** | `hash ... consistent` for session and cache affinity |
| **CDNs** | Mapping content to edge servers |
| **Akka Cluster Sharding** | Actor placement |

⚠️ **Redis Cluster's slot model is worth distinguishing.** 16,384 slots are assigned to nodes explicitly; adding a node means reassigning slots. It achieves the same goal (moving a fraction of data) with an explicit, operator-visible mapping instead of a ring — and it is effectively the **logical shards** idea from [Advanced Partitioning](02-advanced-partitioning.md).

---

## Alternatives worth knowing

### Rendezvous hashing (highest random weight)

```python
def get_node(key, nodes):
    return max(nodes, key=lambda n: hash_combine(key, n))
```

For each key, compute a score with every node and pick the highest.

| ✅ | ❌ |
|---|---|
| No ring, no virtual nodes, simpler | O(N) per lookup instead of O(log N) |
| ✅ Naturally even distribution | Slower with many nodes |
| Trivial to get an ordered list of replicas | |
| Minimal disruption on membership change | |

**Excellent for small N** (tens of nodes) — and genuinely simpler. Worth naming as an alternative.

### Jump consistent hash (Google)

```python
def jump_hash(key: int, num_buckets: int) -> int:
    b, j = -1, 0
    while j < num_buckets:
        b = j
        key = key * 2862933555777941757 + 1
        j = int((b + 1) * ((1 << 31) / float((key >> 33) + 1)))
    return b
```

| ✅ | ❌ |
|---|---|
| O(log N) time, **O(1) memory** — no ring at all | ❌ Buckets must be numbered 0…N−1 |
| Perfectly even distribution | ❌ Can only add/remove the **last** bucket |
| Extremely fast | No arbitrary node removal, no weights |

**Great for a fixed, ordered set of shards** (e.g. logical shard assignment); useless when arbitrary nodes come and go.

### Maglev hashing (Google's load balancer)
Builds a lookup table for O(1) lookups with minimal disruption. Used in high-performance load balancers where per-packet lookup cost matters.

---

## Limitations

⚠️ **Consistent hashing solves *placement*, not everything:**

| It does not solve | Because |
|---|---|
| **Hot keys** | A key still maps to one node; consistent hashing spreads *keys*, not *traffic*. Use caching or salting |
| **Data movement** | It tells you *what* must move; you still have to move it while serving traffic |
| **Range queries** | Hashing destroys ordering — adjacent keys land on different nodes |
| **Heterogeneous load** | Weights help with capacity, not with per-key traffic skew |

☠️ **The range-query loss is a real design constraint.** If you need "all events between two timestamps," hash-based placement is the wrong choice — you need range partitioning (with its hotspot risk) or a composite key with hash-partitioned buckets plus in-partition ordering.

---

## The interview answer

> "I'd use consistent hashing with virtual nodes. With plain modulo hashing, losing one node out of ten remaps about 90% of keys — for a cache that's a total cold start, and for a database it's a full data reshuffle. Consistent hashing moves only about 1/N.
>
> Virtual nodes are essential: with one position per node the distribution is uneven, and more importantly a departing node dumps its entire load onto a single neighbour, which can then fail too. With 150 virtual positions per node, load variance is around 10% and a departing node's keys spread across every survivor. Weights also fall out for free — a bigger machine just gets more virtual nodes.
>
> For replication I'd take the next R distinct physical nodes clockwise, skipping any that share a rack or availability zone, so replicas don't share a failure domain.
>
> The limits: it doesn't solve hot keys — a single popular key still lands on one node, so I'd handle that with a short-TTL local cache or key salting — and it destroys key ordering, so range queries need a different scheme."

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| Explain why modulo hashing moves ~90% of keys | ☐ |
| Describe the ring and the clockwise rule | ☐ |
| Give the three reasons virtual nodes are required | ☐ |
| State the load variance for 1, 10, 100, 500 vnodes | ☐ |
| Implement the ring, including `get_nodes` for replication | ☐ |
| Explain why replicas must be *distinct physical* nodes | ☐ |
| Explain rack/zone awareness | ☐ |
| Distinguish Redis Cluster's slot model | ☐ |
| Compare rendezvous and jump hashing | ☐ |
| Name the four things consistent hashing does *not* solve | ☐ |
| Deliver the interview answer | ☐ |

---

**← Previous** [5.2.2 Advanced Database Partitioning](02-advanced-partitioning.md)
**Next →** [5.2.4 Database Partition Tutorial](04-partition-codelab.md)
