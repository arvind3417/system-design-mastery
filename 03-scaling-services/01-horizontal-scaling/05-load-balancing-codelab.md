# 3.1.5 — Load Balancing Codelab

> **Part 3 · Scaling Services · Horizontal Scaling · Chapter 5 of 6**
> 🧪 **Hands-on lab.** Build a load-balanced fleet, watch algorithms behave differently, break a backend, and prove sticky sessions are a trap.

---

## 🧒 ELI5 — Explain Like I'm 5

You've read about the person at the door. Now **be** the person at the door.

You'll start three tills (servers), put a door person (NGINX) in front, and then:

- Send 100 customers and **count how many went to each till**. Round robin gives 33/33/34. Try other rules and watch the numbers change.
- **Make one till slow on purpose** and see how each rule reacts. (Round robin keeps sending people to the slow till. "Least busy" doesn't. This is the whole argument in one experiment.)
- **Switch a till off** and watch the door person notice — and count exactly how many seconds of errors happen before they do.
- **Turn on "always send this customer to the same till"**, then switch that till off, and watch that customer's shopping basket vanish.

That last one is the lesson. Reading "sticky sessions are risky" is forgettable. Watching your own cart disappear is not.

---

## Setup

```yaml
# docker-compose.yml
services:
  app1: &app
    image: hashicorp/http-echo
    command: ["-text=server-1", "-listen=:5678"]
  app2:
    <<: *app
    command: ["-text=server-2", "-listen=:5678"]
  app3:
    <<: *app
    command: ["-text=server-3", "-listen=:5678"]
  lb:
    image: nginx:alpine
    ports: ["8080:80"]
    volumes: ["./nginx.conf:/etc/nginx/nginx.conf:ro"]
    depends_on: [app1, app2, app3]
```

```nginx
# nginx.conf
events { worker_connections 1024; }
http {
  upstream backend {
    server app1:5678 max_fails=2 fail_timeout=10s;
    server app2:5678 max_fails=2 fail_timeout=10s;
    server app3:5678 max_fails=2 fail_timeout=10s;
  }
  server {
    listen 80;
    location / {
      proxy_pass http://backend;
      proxy_connect_timeout 1s;
      proxy_read_timeout    2s;
      proxy_next_upstream error timeout http_502 http_503;
      add_header X-Upstream $upstream_addr always;
    }
    location /nginx_status { stub_status; }
  }
}
```

```bash
docker compose up -d
curl -s localhost:8080
```

---

## Drill 1 — Round robin distribution

```bash
for i in $(seq 1 300); do curl -s localhost:8080; done | sort | uniq -c
```

**Expected:**
```
100 server-1
100 server-2
100 server-3
```

✅ **Check:** perfectly even. Round robin distributes by *count*, which is correct only when every request costs the same.

---

## Drill 2 — Make one backend slow; watch round robin fail

Replace `app3` with a deliberately slow server:

```yaml
  app3:
    image: python:3.12-alpine
    command: >
      python -c "
      import http.server, time
      class H(http.server.BaseHTTPRequestHandler):
          def do_GET(self):
              time.sleep(2)
              self.send_response(200); self.end_headers()
              self.wfile.write(b'server-3-SLOW')
          def log_message(self,*a): pass
      http.server.HTTPServer(('',5678),H).serve_forever()"
```

```bash
docker compose up -d app3
time (for i in $(seq 1 30); do curl -s -m 5 localhost:8080 >/dev/null; done)
```

✅ **Check:** roughly a third of requests take 2 s. Total time ≈ 20 s instead of ≈ 0.1 s. **Round robin keeps feeding the slow backend because it only counts requests, not cost.**

Now switch to least-connections:

```nginx
upstream backend {
  least_conn;                      # ← add this line
  server app1:5678 max_fails=2 fail_timeout=10s;
  server app2:5678 max_fails=2 fail_timeout=10s;
  server app3:5678 max_fails=2 fail_timeout=10s;
}
```

```bash
docker compose restart lb
time (for i in $(seq 1 30); do curl -s -m 5 localhost:8080 >/dev/null; done)
for i in $(seq 1 90); do curl -s -m 5 localhost:8080; done | sort | uniq -c
```

✅ **Check:** with concurrency, the slow backend accumulates in-flight connections and receives **noticeably fewer** requests. This is the practical argument for least-connections over round robin, and you just measured it.

⚠️ Note the caveat: with strictly *sequential* requests, connections never pile up, so `least_conn` looks identical to round robin. Run with concurrency to see the effect:

```bash
seq 1 200 | xargs -P 20 -I{} curl -s -m 5 localhost:8080 | sort | uniq -c
```

---

## Drill 3 — Kill a backend, measure the error window

```bash
# Continuous traffic in one terminal
while true; do
  curl -s -m 2 -o /dev/null -w "%{http_code} " localhost:8080
  sleep 0.2
done
```

```bash
# In another terminal
docker compose stop app2
```

✅ **Check:** you see a small number of `502`s, then clean `200`s. Count them.

**Why any errors at all?** NGINX's passive health checking marks a backend down only after `max_fails=2` failures. Those two failures are real user-facing errors.

**How to reduce the window:**
1. `proxy_next_upstream` (already set) — NGINX retries the *same* request on another backend, so many failures are invisible to the client. Verify by removing it and re-running.
2. Active health checks (NGINX Plus, or Envoy/HAProxy in open source) detect death *before* a user request hits it.
3. Graceful shutdown: deregister → wait → drain, so planned removals cause **zero** errors.

```bash
docker compose start app2
```
✅ **Check:** after `fail_timeout=10s`, traffic returns to app2 automatically.

---

## Drill 4 — Sticky sessions and the vanishing cart

```nginx
upstream backend {
  ip_hash;                          # sticky by client IP
  server app1:5678;
  server app2:5678;
  server app3:5678;
}
```

```bash
docker compose restart lb
for i in $(seq 1 20); do curl -s localhost:8080; done | sort | uniq -c
```

✅ **Check:** **all 20 requests go to one backend.** Your client IP hashes to exactly one.

Now the lesson:

```bash
# Find which backend you're pinned to, then stop it
docker compose stop app1     # (or whichever you were pinned to)
for i in $(seq 1 5); do curl -s localhost:8080; done
```

✅ **Check:** you are rehashed to a different backend. In a real app, **everything that server held in memory for you — your session, your cart, your half-completed form — is gone.**

**Also observe the distribution problem.** Simulate several clients:
```bash
for ip in 1 2 3 4 5 6 7 8 9 10; do
  curl -s -H "X-Forwarded-For: 10.0.0.$ip" localhost:8080
done | sort | uniq -c
```
(With `ip_hash` on the real source IP this won't vary in Docker — the point stands conceptually: hashing thousands of NAT'd users by IP produces badly uneven load.)

🎯 **Takeaway:** stickiness is fine as a *cache-locality optimisation*. It is never a substitute for externalised session state.

---

## Drill 5 — Weighted routing (canary deploys)

```nginx
upstream backend {
  server app1:5678 weight=9;       # 90% — stable
  server app2:5678 weight=9;       # (two stable instances)
  server app3:5678 weight=2;       # ~10% — canary
}
```

```bash
docker compose restart lb
for i in $(seq 1 200); do curl -s localhost:8080; done | sort | uniq -c
```

✅ **Check:** roughly 90/10 split. **This is a canary deploy in three characters of config.** In production you'd pair it with automated rollback triggered by the canary's error rate.

---

## Drill 6 — Timeout alignment

Set a deliberately misaligned configuration:

```nginx
proxy_read_timeout 1s;      # LB gives up after 1s
```

with `app3` sleeping 2 s.

```bash
docker compose restart lb
curl -s -w "\n%{http_code} in %{time_total}s\n" localhost:8080
```

✅ **Check:** `504 Gateway Timeout` after 1 s — even though the backend would have answered successfully at 2 s. The LB gave up on healthy work.

Now the opposite error: set `proxy_read_timeout 60s` and have the client use `curl -m 2`.

✅ **Check:** the client gives up at 2 s, but **the backend keeps working for the full 2 s (and would for 60 s)**, producing an answer nobody reads. This is the deadline-propagation waste from [Chapter 2.1.5](../../02-microservices-and-dataflow/01-synchronous-communication/05-timeout.md).

**Correct ordering:** `client timeout > LB timeout > backend timeout`, each with a small margin.

---

## Drill 7 — Connection multiplexing

```bash
# Watch connection counts
curl -s localhost:8080/nginx_status

# Hammer with many concurrent clients
seq 1 500 | xargs -P 100 -I{} curl -s -o /dev/null localhost:8080
curl -s localhost:8080/nginx_status
```

**Expected output shape:**
```
Active connections: 100
server accepts handled requests
 500 500 500
Reading: 0 Writing: 1 Waiting: 99
```

✅ **Check:** NGINX holds ~100 client connections but uses **far fewer** upstream connections (enable `keepalive 32;` in the upstream block to see this clearly). This is what protects backend connection limits — the single most common non-obvious benefit of a reverse proxy.

```nginx
upstream backend {
  least_conn;
  server app1:5678;
  server app2:5678;
  server app3:5678;
  keepalive 32;                     # reuse upstream connections
}
server {
  location / {
    proxy_pass http://backend;
    proxy_http_version 1.1;         # required for keepalive
    proxy_set_header Connection ""; # required for keepalive
  }
}
```

---

## Drill 8 — Consistent hashing for cache locality

```nginx
upstream cache_backend {
  hash $request_uri consistent;     # same URI → same backend
  server app1:5678;
  server app2:5678;
  server app3:5678;
}
```

```bash
docker compose restart lb
for i in 1 2 3; do
  for p in /a /b /c /d /e; do
    curl -s "localhost:8080$p" ;
  done
done | paste - - - - -
```

✅ **Check:** each path consistently lands on the same backend across all three rounds. That's cache locality: backend 2 always serves `/b`, so its local cache for `/b` is always warm.

Now remove a backend and observe:
```bash
docker compose stop app2
for p in /a /b /c /d /e; do curl -s "localhost:8080$p"; done
```
✅ **Check:** with `consistent`, only the keys that mapped to `app2` move. Without the `consistent` keyword (plain modulo hashing), **most** keys would move. That difference is the entire value of consistent hashing — see [Consistent Hashing](../../05-scaling-data-storage/02-data-partitioning/03-consistent-hashing.md).

---

## Drill 9 — Slow start

Add a brand-new backend to a busy fleet with no ramp-up, and it receives its full share of traffic immediately — with a cold cache, cold JIT, and empty connection pools. It gets slow, fails health checks, is removed, restarts, and the cycle repeats.

```nginx
server app4:5678 slow_start=30s;    # NGINX Plus; Envoy has slow_start_config
```

✅ **Reason about it:** with slow start, traffic ramps over 30 s, letting caches warm. Without it, an autoscaling event during a traffic spike can make things *worse* — the classic "scaling made it slower" incident.

---

## 📋 Lab checklist

| Drill | Done? | Takeaway |
|---|---|---|
| 1 Round robin | ☐ | Even by count, not by cost |
| 2 Slow backend | ☐ | Least-connections adapts, round robin doesn't |
| 3 Backend failure | ☐ | Passive checks cost real errors; `proxy_next_upstream` hides many |
| 4 Sticky sessions | ☐ | Uneven load and lost state on failure |
| 5 Weighted routing | ☐ | Canary deploys are a weight change |
| 6 Timeouts | ☐ | Must be ordered client > LB > backend |
| 7 Keepalive | ☐ | The proxy protects backend connection limits |
| 8 Consistent hashing | ☐ | Only 1/N of keys move on a node change |
| 9 Slow start | ☐ | Cold instances need a ramp |

---

**← Previous** [3.1.4 Load Balancer](04-load-balancer.md)
**Next →** [3.1.6 Auto Scaling](06-auto-scaling.md)
