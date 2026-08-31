# 2.1.6 — Retries

> **Part 2 · Microservices & Data Flow · Synchronous Communication · Chapter 6 of 9**
> The technique that fixes transient failures — and, done wrong, causes outages.

---

## 🧒 ELI5 — Explain Like I'm 5

The line was noisy, so you call again. Sensible.

But here's what goes wrong when **everyone** does it:

The dessert kitchen gets a bit slow. So all 50 cooks call again. And again. Now the dessert kitchen has **150 calls** instead of 50 — and it was already struggling. It gets slower. Everyone retries more. It dies completely.

**You didn't help. You finished it off.** That's a *retry storm*.

Three rules stop this:

1. **Wait longer each time.** Try after 1 second, then 2, then 4. Don't hammer. *(Exponential backoff.)*
2. **Everyone waits a slightly different amount.** If all 50 cooks wait exactly 1 second, they all call again at the exact same moment — you've just moved the stampede, not stopped it. So each waits a *random* amount. *(Jitter.)*
3. **Cap the total.** "No more than 1 extra call for every 10 normal ones, across the whole kitchen." *(Retry budget.)*

And one more thing, the most important: **only retry if repeating it is harmless.** Calling again to *ask* something is fine. Calling again to *order 20 desserts* might get you 40 desserts.

---

## When to retry

| Error class | Retry? | Why |
|---|---|---|
| Connection refused / reset | ✅ Yes | Server restarting or instance replaced |
| DNS resolution failure | ✅ Yes | Transient resolver blip |
| 503 Service Unavailable / `UNAVAILABLE` | ✅ Yes | Explicitly transient |
| 429 Too Many Requests | ✅ Yes, after `Retry-After` | Backpressure signal |
| 502 / 504 from a proxy | ⚠️ Cautiously | The request may have reached the server |
| Timeout (`DEADLINE_EXCEEDED`) | ⚠️ **Only if idempotent** | Unknown whether the work happened |
| 500 Internal Server Error | ⚠️ Cautiously, with backoff | Might be a deterministic bug — infinite retry won't help |
| 400 / 422 Bad Request | ❌ Never | The request is wrong; retrying repeats the bug |
| 401 / 403 | ❌ Never (unless refreshing a token once) | |
| 404 | ❌ Never | |
| 409 Conflict | ❌ Usually | Often the correct outcome of an idempotent retry |
| `ABORTED` (optimistic concurrency) | ✅ Yes | Retry the whole operation, re-reading state |

🎯 **The rule to state in an interview:** *"Retry only transient errors, and only for idempotent operations — or for non-idempotent ones carrying an idempotency key."*

---

## The safety condition: idempotency

**A retry is safe only if repeating the operation has the same effect as doing it once.**

| Operation | Idempotent? |
|---|---|
| `GET /users/5` | ✅ Inherently |
| `PUT /users/5` (full replace) | ✅ Inherently |
| `DELETE /users/5` | ✅ In effect (second call returns 404/204) |
| `POST /payments` | ❌ **No** — double charge |
| `POST /payments` with `Idempotency-Key` | ✅ Made safe |
| `UPDATE counter SET n = n + 1` | ❌ No |
| `UPDATE counter SET n = 42` | ✅ Yes |
| `INSERT` with a natural unique key | ✅ Effectively (second attempt conflicts) |
| Sending an email | ❌ No — dedupe by a message key |

**Making things idempotent** — three techniques:

1. **Idempotency key** — client generates a UUID; server stores `key → response` and replays it. (Implementation in [Chapter 2.1.3](03-implementation.md#idempotency-on-the-receiving-side).)
2. **Natural idempotency** — design operations as *state assignments* (`set status = shipped`) rather than *deltas* (`increment`). Deltas are never idempotent.
3. **Deduplication window** — the receiver keeps recently-seen message IDs for N minutes and drops repeats. Cheap, but only correct within the window.

---

## Backoff and jitter

### Why exponential backoff

A struggling service needs **less** load, not the same load repeated. Exponential backoff reduces pressure geometrically while still recovering quickly when the problem is brief.

```
attempt 1: fail → wait  100 ms
attempt 2: fail → wait  200 ms
attempt 3: fail → wait  400 ms
attempt 4: fail → wait  800 ms   (capped at, say, 5 s)
```

$$\text{delay} = \min(\text{cap},\; \text{base} \times 2^{\text{attempt}})$$

### Why jitter is mandatory

Without jitter, all clients that failed at the same instant retry at the same instant. You have converted a continuous overload into a series of **synchronised spikes** — often worse.

```
No jitter:            ████░░░░████░░░░████     (thundering herd)
Full jitter:          ▂▃▁▂▄▁▃▂▁▄▂▃▁▂▃▁▄▂       (smooth)
```

Variants (AWS's "Exponential Backoff and Jitter" is the canonical reference):

| Variant | Formula | Notes |
|---|---|---|
| No jitter | `delay = base × 2^n` | ❌ Synchronised herds |
| Full jitter | `delay = random(0, base × 2^n)` | ✅ **Best general choice** — lowest contention |
| Equal jitter | `delay = half + random(0, half)` where `half = base × 2^n / 2` | Slightly more predictable |
| Decorrelated jitter | `delay = min(cap, random(base, prev × 3))` | ✅ Excellent for high contention |

```python
import random

def full_jitter_delay(attempt, base_ms=100, cap_ms=5000):
    return random.uniform(0, min(cap_ms, base_ms * (2 ** attempt)))
```

---

## Retry budgets and amplification

### The amplification problem

```
Service A calls B calls C.
Each layer retries 3 times.
One user request → 3 (A→B) × 3 (B→C) = 9 calls to C.
With four layers: 3⁴ = 81 calls.
```

☠️ **Nested retries multiply.** During an incident, a 3-layer stack with 3 retries each turns 1,000 QPS of user traffic into 27,000 QPS at the bottom — precisely when the bottom is least able to cope.

### The fixes

**1. Retry at one layer only.** Usually the layer closest to the failure, or the edge. Explicitly disable retries in the middle. This is the single most effective rule.

**2. Retry budgets.** Cap retries as a *fraction of successful traffic*, globally, not per request.

```
Budget: retries ≤ 10% of successful requests over a 10 s window.
Steady state (1% failures): plenty of budget, retries help.
Incident (80% failures):    budget exhausted in milliseconds → retries stop.
```

This is elegant: retries help exactly when they can help (isolated failures) and automatically stop when they cannot (systemic failure). Envoy, gRPC, and Finagle implement this; it is a strong thing to name.

**3. Circuit breakers** — the coarse version of the same idea. ([Next chapter](07-circuit-breaker.md).)

**4. Cap attempts low.** 2–3 total attempts. More than 3 almost never helps: if two retries with backoff failed, the problem is not transient.

---

## Retry configuration, worked

```yaml
retry:
  max_attempts: 3                 # 1 initial + 2 retries
  per_attempt_timeout: 500ms
  backoff:
    base: 100ms
    multiplier: 2
    max: 2s
    jitter: full
  retry_on: [connect_failure, refused_stream, unavailable, 503, 429]
  budget:
    max_retry_ratio: 0.10         # ≤10% of successful requests
    min_retries_per_second: 3     # allow a floor so low-traffic services can retry
  respect_retry_after: true
```

**Total worst-case latency:**
`500 + jitter(0..100) + 500 + jitter(0..200) + 500 ≈ 1,800 ms`
The caller's deadline must exceed this, or reduce `max_attempts`.

---

## Retrying the right target

A retry to the **same failed instance** is usually pointless. Combine retries with:

- **Load-balancer re-pick** — each attempt selects a different healthy instance.
- **Outlier ejection** — the failing instance is removed from the pool automatically.
- **Hedging** (a different technique, don't confuse them): send a *duplicate* request to a second replica after p95 elapses, without waiting for a failure. Cuts p99 dramatically at ~5% extra load, and requires idempotency. Retries fix errors; hedging fixes latency.

---

## Client-side vs server-side retry behaviour

**Servers can guide clients.** Use `Retry-After` on 429 and 503:

```http
503 Service Unavailable
Retry-After: 12
```

Well-behaved clients honour it, which lets a struggling service **control its own recovery**. This is one of the few mechanisms by which a server can protect itself from its clients.

For queues and async work, retries are the broker's job: visibility timeouts, redelivery, and a **dead-letter queue** after N attempts. Never build an unbounded retry loop in a consumer — poison messages will retry forever and block the partition.

---

## ☠️ Failure modes

| Mistake | Consequence |
|---|---|
| Retrying non-idempotent writes | Duplicate charges, duplicate orders, duplicate emails |
| Retrying 4xx errors | Infinite loop on a permanent condition |
| No jitter | Synchronised retry spikes |
| No backoff | Pure load amplification |
| Nested retries at every layer | 3ⁿ amplification |
| No retry budget | Retries make every incident worse |
| Retry attempts inside a too-short deadline | Retries never actually execute |
| Retrying to the same dead instance | Guaranteed failure, wasted budget |
| Unbounded consumer retries on a poison message | The partition stalls forever |
| Ignoring `Retry-After` | You prevent the service from recovering |

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| Classify ten error types as retryable or not | ☐ |
| State the idempotency safety condition | ☐ |
| Give three ways to make an operation idempotent | ☐ |
| Explain why jitter is mandatory, with the spike diagram | ☐ |
| Write the full-jitter formula | ☐ |
| Explain 3ⁿ retry amplification and three fixes | ☐ |
| Explain retry budgets and why they self-disable | ☐ |
| Distinguish retries from hedged requests | ☐ |
| Compute worst-case latency for a retry config | ☐ |

---

**← Previous** [2.1.5 Timeout](05-timeout.md)
**Next →** [2.1.7 Circuit Breaker](07-circuit-breaker.md)
