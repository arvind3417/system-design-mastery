# 1.2.3 — API Design: Pagination

> **Part 1 · Introduction · API Design · Chapter 3 of 6**
> A small topic that reveals whether you have operated a large database.

---

## 🧒 ELI5 — Explain Like I'm 5

You have a phone book with ten million names and someone asks for "the next twenty names."

**The lazy way:** *"Give me names 1,000,001 to 1,000,020."* To find name one million, the librarian has to **count past a million names first**, every single time. The further you go, the slower it gets. And if someone adds a new name at the front while you're reading, everything shifts by one and you see the same name twice.

**The smart way:** you remember the **last name you read** — "I stopped at *Patel, Riya*" — and next time you say *"start just after Patel, Riya."* The librarian opens straight to that spot. Page 500,000 is exactly as fast as page 1. And new names appearing at the front don't shift your place.

That's it. The lazy way is **offset pagination**. The smart way is **cursor pagination**. At scale, always use the smart way.

---

## The three strategies

| Strategy | Request | Cost of page N | Stable under writes? | Can jump to page N? |
|---|---|---|---|---|
| **Offset / page number** | `?page=5&size=20` or `?offset=100&limit=20` | O(offset + limit) — grows linearly | ❌ No | ✅ Yes |
| **Cursor / keyset** | `?after=<cursor>&limit=20` | O(limit) — constant | ✅ Yes | ❌ No |
| **Seek by known key** | `?since_id=12345&limit=20` | O(limit) | ✅ Yes | Partially |

---

## Offset pagination — why it breaks

```sql
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 1000000;
```

The database must **produce and discard one million rows** to return twenty. There is no index structure that avoids this for a general offset; the rows must be counted.

🔢 **Measured reality on a ~10M-row Postgres table with an index on `created_at`:**

| Offset | Approx latency |
|---|---|
| 0 | 1 ms |
| 10,000 | 15 ms |
| 100,000 | 130 ms |
| 1,000,000 | 1.2 s |
| 5,000,000 | 6 s (and it's now a scan) |

Two independent failures follow:

**1. Cost.** Deep pages become expensive enough that a crawler walking to page 50,000 constitutes an accidental denial-of-service on your own database.

**2. Correctness — the shifting window.** New rows arrive at the head while a client paginates:

```
t0  page 1 = [P10 P9 P8 ... P1]         (offset 0, limit 10)
t1  someone posts P11
t2  page 2 = offset 10 → [P1 ...]       ← P1 was already on page 1: DUPLICATE
```

Deletions cause the mirror-image bug: **skipped items**. Users see the same post twice, or never see a post at all, and this is essentially impossible to reproduce in a bug report.

### When offset is still fine

- Admin tools over small tables (< 10k rows).
- Anything where the user genuinely needs "jump to page 47."
- Total-count displays where the set is bounded.

**Mitigation if you must:** cap the maximum offset (`offset > 10000 → 400 Bad Request`, "use cursor pagination or narrow your filter"). Most large public APIs do exactly this.

---

## Cursor (keyset) pagination — how it actually works

Instead of "skip N rows", you say "give me rows **after this exact position** in the sort order."

```sql
-- First page
SELECT * FROM posts
ORDER BY created_at DESC, id DESC
LIMIT 20;

-- Next page: last row of previous page was (created_at='2026-08-31T09:00:00Z', id=8842)
SELECT * FROM posts
WHERE (created_at, id) < ('2026-08-31T09:00:00Z', 8842)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

With an index on `(created_at DESC, id DESC)` the database **seeks directly** to the position and reads 20 rows. Constant cost, any depth.

### The tuple comparison is not optional

`WHERE created_at < $1` alone is **wrong**: if two posts share a timestamp, you skip or duplicate them at the page boundary. The row comparison `(created_at, id) < ($1, $2)` — supported natively in Postgres/MySQL, expandable manually elsewhere — makes the sort key **total** (unique), which is what makes the cursor unambiguous.

> **Rule: a cursor requires a total ordering.** Always append a unique tiebreaker (the primary key) to the sort key.

Manual expansion for engines without row comparison:

```sql
WHERE created_at < $1
   OR (created_at = $1 AND id < $2)
```

### Cursor encoding

The cursor should be **opaque** — clients must not parse or construct it.

```
raw:     {"created_at":"2026-08-31T09:00:00Z","id":8842,"dir":"next","v":1}
base64:  eyJjcmVhdGVkX2F0IjoiMjAyNi0wOC0zMVQwOTowMDowMFoiLCJpZCI6ODg0MiwiZGlyIjoibmV4dCIsInYiOjF9
```

Include:
- the sort-key values (the position),
- a **version** field so you can change the cursor format later,
- optionally a **signature (HMAC)** so clients can't forge cursors that make you scan arbitrary ranges,
- optionally the **filter set hash**, so a cursor from one filter can't be replayed against another.

☠️ **Failure mode:** exposing `?after=8842` as a bare ID. Clients hard-code arithmetic on it, and now you can never change the sort order. Always base64 an opaque struct.

### Response shape

```json
{
  "items": [ /* 20 items */ ],
  "page_info": {
    "next_cursor": "eyJjcmVhdGVkX2F0Ijoi...",
    "prev_cursor": "eyJjcmVhdGVkX2F0Ijoi...",
    "has_next": true,
    "has_prev": false
  }
}
```

**Trick for `has_next` without a `COUNT`:** request `limit + 1` rows. If you get `limit + 1` back, there is a next page — return the first `limit` and set `has_next: true`. One row of extra work instead of a full count.

---

## What cursor pagination gives up

| Lost capability | Workaround |
|---|---|
| "Jump to page 500" | Usually not a real product requirement. Offer filters and search instead. |
| Exact total count | Return an approximate count (`reltuples`, a cached estimate, or HyperLogLog) or none at all. Exact counts on big tables are expensive anyway. |
| Arbitrary sort by any column | Every sortable column needs its own index including the tiebreaker |
| Random access by external systems | Provide a bulk export endpoint instead |

⚖️ **Trade-off summary:** cursor pagination trades *random access* for *constant cost and correctness*. For feeds, timelines, logs, and infinite scroll — which is nearly everything at scale — that trade is obviously right.

---

## Special cases

### Bidirectional pagination
Support both `after` and `before`. `before` runs the same query with the comparison and ORDER BY reversed, then reverses the result set in the application before returning it.

### Sorting by a mutable field
If you paginate by `updated_at` and a row is updated mid-pagination, it **moves** — it can appear on a later page again. If duplicates are unacceptable, either sort by an immutable field (`created_at`, `id`) or deduplicate client-side by ID. Say this out loud; it's a subtle point that impresses.

### Pagination across shards
The problem: page ordering is global, data is split across N shards.

Approach: query each of the N shards for `limit` rows past its own per-shard cursor, merge-sort the N result sets in the application, return the top `limit`, and encode **all N per-shard positions** into one composite cursor.

Cost: you read `N × limit` rows to return `limit`. This is the *scatter-gather* tax. With 100 shards and `limit=20` you read 2,000 rows per page. Mitigations: reduce the fan-out by routing on the filter (if the query includes `user_id`, only one shard is involved), or maintain a denormalised, pre-sorted per-user timeline ([Pre-Computing Pattern](../../07-patterns-and-templates/01-patterns/03-pre-computing-pattern.md)).

### Ranked / personalised feeds
When ordering is by a score that is recomputed per request, no stable cursor exists. Standard solution: **materialise the ranked list once per session** into a cache (`session:{id}:feed → [id1, id2, ...]`), then paginate over that frozen list by index. The cursor becomes `(session_id, offset)` into a fixed array — offset pagination over an in-memory list, which is O(1).

### Infinite scroll on mobile
Cursor pagination is a perfect fit. Additionally: return `limit` between 20 and 50, prefetch page N+1 when the user reaches 70% of page N, and make the cursor durable across app restarts so scroll position survives.

---

## Interview script

> "I'll use cursor pagination, not offset. Offset is O(offset) in the database, so deep pages get slow and expensive, and it produces duplicates and skips when items are inserted while a client is paginating. The cursor encodes `(created_at, id)` — I need the ID as a tiebreaker so the sort key is a total order, otherwise rows sharing a timestamp get skipped at the page boundary. I'll base64 an opaque struct with a version field so I can evolve the format, and I'll fetch `limit + 1` rows to compute `has_next` without a COUNT. The cost is that we lose 'jump to page N' and exact totals — for an infinite-scroll feed that's the right trade."

Roughly 45 seconds, and it covers correctness, performance, evolution, and trade-off.

---

## 📋 Chapter checklist

| Concept | Ready? |
|---|---|
| Explain why `OFFSET 1000000` is slow | ☐ |
| Draw the duplicate/skip bug with a timeline | ☐ |
| Write the keyset SQL with a tuple comparison | ☐ |
| Explain why the tiebreaker is mandatory | ☐ |
| Compute `has_next` without a COUNT | ☐ |
| Explain cross-shard pagination and its cost | ☐ |
| Explain how ranked feeds are paginated | ☐ |

---

**← Previous** [1.2.2 API Design Example](02-api-design-example.md)
**Next →** [1.2.4 API Authentication](04-api-authentication.md)
