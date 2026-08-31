# 2.1.3 — Implementing Synchronous Communication

> **Part 2 · Microservices & Data Flow · Synchronous Communication · Chapter 3 of 9**
> From protocol choice to connection pools to contracts. The engineering details.

---

## 🧒 ELI5 — Explain Like I'm 5

You've decided the kitchens will use phones. Now the boring-but-vital questions:

- **What language do we speak on the phone?** English? Code words? Short code words are faster but nobody outside understands them. *(JSON vs Protobuf.)*
- **Do we hang up after every call?** Dialling takes ages. Better to **keep the line open** and reuse it. *(Connection pooling and keep-alive.)*
- **How do we agree what a message means?** If the pizza kitchen suddenly renames "large" to "XL", the dessert kitchen gets confused. So we write down the agreed words **before** we change them. *(Schemas and contracts.)*
- **How many lines do we need?** Too few and cooks queue for the phone. Too many and the switchboard melts. *(Pool sizing.)*
- **How do we find the number?** Kitchens move. You need an up-to-date phone book. *(Service discovery.)*

None of this is glamorous. All of it decides whether the system works at 3 a.m.

---

## Protocol: REST/JSON vs gRPC/Protobuf

| | REST + JSON | gRPC + Protobuf |
|---|---|---|
| Encoding | Text | Binary |
| Payload size | Baseline | 3–10× smaller |
| CPU to encode/decode | Baseline | 2–10× cheaper |
| Transport | HTTP/1.1 or 2 | HTTP/2 (multiplexed) |
| Contract | OpenAPI (optional, often stale) | `.proto` (compiled, enforced) |
| Streaming | SSE / chunked / WebSocket | Native, four modes |
| Deadlines | Manual header | Built into the protocol |
| Browser support | Native | Needs grpc-web + proxy |
| Debuggability | `curl` and read it | Needs `grpcurl` / reflection |
| Load balancing | Per-request (easy with L7 LB) | Per-connection — **needs care** |

⚠️ **The gRPC load-balancing trap.** gRPC multiplexes many requests over one long-lived HTTP/2 connection. A normal L4 load balancer balances *connections*, so one client pins to one server and stays there — you get terrible distribution. Fixes: client-side load balancing with service discovery, an L7 proxy that understands HTTP/2 (Envoy), or `maxConnectionAge` on the server so connections periodically rebalance.

### Protobuf contract example

```protobuf
syntax = "proto3";
package orders.v1;

service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order);
  rpc ListOrders(ListOrdersRequest) returns (stream Order);   // server streaming
}

message GetOrderRequest {
  string order_id = 1;
}

message Order {
  string order_id   = 1;
  string customer_id = 2;
  int64  total_minor = 3;
  string currency    = 4;
  Status status      = 5;
  reserved 6;                     // a removed field — never reuse the number
  enum Status {
    STATUS_UNSPECIFIED = 0;       // always have a zero default
    PENDING            = 1;
    PAID               = 2;
    CANCELLED          = 3;
  }
}
```

**Schema evolution rules (these are the interview-relevant part):**

| Change | Safe? |
|---|---|
| Add a new optional field with a new number | ✅ Yes |
| Remove a field | ✅ If you `reserved` the number and name |
| Rename a field | ✅ Wire format uses numbers, not names — but breaks JSON mapping |
| **Change a field's number** | ❌ Breaks everything |
| **Change a field's type** | ❌ Breaks everything (except a few compatible int widenings) |
| Add an enum value | ⚠️ Old clients see the unknown value — always have an `UNSPECIFIED = 0` and handle unknowns |
| Make an optional field required | ❌ Breaking |

**The golden rule of contracts: additive changes only.** Both sides must tolerate fields they don't know about. This is what allows independent deployment — the entire point of microservices.

---

## Connection management

### Why it dominates

Opening a TCP+TLS connection costs 2–3 round trips. Cross-AZ that's ~5 ms; cross-region ~200 ms. Doing that per request is often more expensive than the request itself.

### Connection pooling

```
Pool size (Little's Law):  connections = target_QPS × latency_seconds

  1,000 QPS × 0.05 s = 50 connections
  1,000 QPS × 0.50 s = 500 connections   ← latency ×10 → pool ×10
```

That second line is the whole story of how a slow dependency exhausts a pool.

| Setting | Guidance |
|---|---|
| `maxIdleConns` / `minIdle` | Keep enough warm to absorb a burst without new handshakes |
| `maxConnsPerHost` | Bound it — an unbounded pool converts a slow dependency into OOM |
| `idleConnTimeout` | Shorter than any intermediary's idle timeout (LBs commonly close at 60 s) |
| `maxConnAge` | Rotate connections so DNS changes and rebalancing take effect |
| Acquire timeout | **Must exist** — waiting forever for a connection is an unbounded queue |

☠️ **The idle-timeout mismatch bug:** your client keeps connections for 300 s, but the load balancer silently closes them at 60 s. The client picks a dead connection and gets a confusing `EOF`/`connection reset` on the *first* request. Fix: client idle timeout < LB idle timeout, and retry-on-idempotent-connection-error.

### HTTP/2 and multiplexing
One connection carries many concurrent streams, so pool sizing changes from "connections" to "concurrent streams per connection" (`maxConcurrentStreams`, typically 100–250). Fewer connections, but watch the per-connection stream limit — exceeding it silently queues requests.

---

## Serialisation choices

| Format | Size | Speed | Schema | Human-readable |
|---|---|---|---|---|
| JSON | Baseline | Baseline | Optional (JSON Schema) | ✅ |
| MessagePack / CBOR | ~0.7× | 2× | Optional | ❌ |
| Protobuf | ~0.3× | 5× | Required | ❌ |
| Avro | ~0.3× | 5× | Required, schema travels or is registry-resolved | ❌ |
| FlatBuffers / Cap'n Proto | ~0.3× | Zero-copy read | Required | ❌ |

**Rule of thumb:** JSON at the edge (debuggability wins), binary internally (efficiency wins). Avro is the standard for Kafka payloads because of schema-registry integration and its strong evolution rules.

---

## Idempotency on the receiving side

Because the caller *will* retry, and you cannot stop it.

```python
def create_payment(idempotency_key, amount, account_id):
    # 1. Try to claim the key atomically
    claimed = db.execute(
        "INSERT INTO idempotency (key, status, created_at) "
        "VALUES (?, 'in_progress', now()) ON CONFLICT DO NOTHING",
        idempotency_key)

    if not claimed:
        rec = db.fetch("SELECT status, response FROM idempotency WHERE key = ?",
                       idempotency_key)
        if rec.status == 'completed':
            return rec.response          # replay the original response
        raise Conflict409("request in progress")   # caller retries later

    # 2. Do the work and store the result in ONE transaction
    with db.transaction():
        payment = charge(account_id, amount)
        db.execute("UPDATE idempotency SET status='completed', response=? "
                   "WHERE key = ?", serialize(payment), idempotency_key)
    return payment
```

Details that matter:
- The claim must be **atomic** (unique constraint or `INSERT ... ON CONFLICT`), or two concurrent retries both proceed.
- The result must be stored **in the same transaction** as the effect, or a crash leaves the effect done and the key un-completed → the retry does it twice.
- Keys need a **TTL** (24 h is typical) or the table grows forever.
- Return the **original response**, not a fresh one, so the client sees consistent data.
- Scope the key to the caller (`(client_id, key)`) so one tenant can't collide with another.

---

## Error handling and status mapping

| gRPC code | HTTP | Retryable? | Meaning |
|---|---|---|---|
| `OK` | 200 | — | Success |
| `INVALID_ARGUMENT` | 400 | ❌ | Client bug — retrying repeats the bug |
| `UNAUTHENTICATED` | 401 | ❌ (unless refreshing a token) | |
| `PERMISSION_DENIED` | 403 | ❌ | |
| `NOT_FOUND` | 404 | ❌ | |
| `ALREADY_EXISTS` | 409 | ❌ | Often the *correct* outcome of an idempotent retry |
| `FAILED_PRECONDITION` | 412 | ❌ | State is wrong; retrying won't help |
| `ABORTED` | 409 | ✅ | Concurrency conflict — retry the whole operation |
| `RESOURCE_EXHAUSTED` | 429 | ✅ with backoff | Rate limited |
| `UNAVAILABLE` | 503 | ✅ | Transient — the canonical retryable error |
| `DEADLINE_EXCEEDED` | 504 | ⚠️ Only if idempotent | Unknown whether the work happened |
| `INTERNAL` / `UNKNOWN` | 500 | ⚠️ Cautiously | Could be a bug, could be transient |

🎯 **Classify every error as retryable or not, explicitly.** Retrying a 400 forever is a self-inflicted denial-of-service; not retrying a 503 turns a blip into a user-visible failure.

---

## Contract testing

Integration environments with every service running are slow, flaky, and expensive. **Consumer-driven contract testing** (Pact and similar) is the standard replacement:

1. The **consumer** writes a test declaring what it sends and what it expects back.
2. That produces a **contract** file, published to a broker.
3. The **provider's** CI replays every consumer's contract against the real provider.
4. If the provider breaks any consumer's expectation, **the provider's build fails** — before deploy.

This gives independent deployability with confidence, which is the thing microservices promise and usually fail to deliver.

---

## Observability for synchronous calls

**Propagate a trace context on every hop.** W3C `traceparent` is the standard:

```http
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
             │  └ trace id (whole request)        └ span id (this hop)
             └ version
```

Emit **per-dependency** metrics, not just per-service:

```
rpc_client_requests_total{target="payments", code="OK"}
rpc_client_duration_seconds_bucket{target="payments", le="0.1"}
rpc_client_inflight{target="payments"}
circuit_breaker_state{target="payments"}          # 0 closed, 1 half-open, 2 open
connection_pool_available{target="payments"}
connection_pool_wait_seconds{target="payments"}
```

☠️ Without per-dependency metrics, an incident looks like "service A is slow" and you spend 40 minutes finding out it was C all along. With them, the dashboard says it immediately. This is directly MTTR, which is directly availability.

---

## A complete client, conceptually

```
Request
  → deadline check (is there time left?)
  → circuit breaker (is this dependency open?)
  → bulkhead (acquire from THIS dependency's pool, with a wait timeout)
  → load balancer pick (from service discovery, with outlier ejection)
  → connection from pool (keep-alive, HTTP/2)
  → serialise (Protobuf) + inject traceparent + deadline header
  → send, with a per-attempt timeout < remaining deadline
  → on retryable error: backoff + jitter, if retry budget allows and op is idempotent
  → on final failure: fallback (cache / default / degraded)
  → record metrics, close the span
```

Modern stacks give you most of this: a **service mesh** (Envoy sidecar) provides discovery, load balancing, retries, timeouts, circuit breaking, mTLS, and metrics with no application code. A **library** (Resilience4j, Polly, gRPC interceptors) does the same in-process. **Do not hand-roll this.**

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| Compare REST/JSON and gRPC on six dimensions | ☐ |
| Explain the gRPC load-balancing trap and its fixes | ☐ |
| State the additive-only schema evolution rule | ☐ |
| Size a connection pool with Little's Law | ☐ |
| Explain the idle-timeout mismatch bug | ☐ |
| Write an idempotent handler with atomic key claiming | ☐ |
| Classify eight error codes as retryable or not | ☐ |
| Explain consumer-driven contract testing | ☐ |
| List the per-dependency metrics you'd emit | ☐ |

---

**← Previous** [2.1.2 Synchronous Communication](02-synchronous-communication.md)
**Next →** [2.1.4 Failure Handling in Synchronous Communication](04-failure-handling.md)
