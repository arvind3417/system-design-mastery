# 3.1.1 — Evolution of Computing Environments

> **Part 3 · Scaling Services · Horizontal Scaling · Chapter 1 of 6**
> Why modern architecture looks the way it does — the constraints that produced it.

---

## 🧒 ELI5 — Explain Like I'm 5

Imagine how people got computers over the years:

1. **One enormous computer in a room**, and everyone shares it by taking turns. It costs as much as a house. *(Mainframe.)*
2. **Everyone gets their own small computer**, and they're all wired together. Cheaper, and if one breaks, only one person is sad. *(Client-server.)*
3. **You buy computers and put them in a special building** with air conditioning and backup power. Buying one takes six weeks. *(Your own data centre.)*
4. **One physical computer pretends to be ten computers.** Much less waste — you were only using 5% of that machine anyway. *(Virtual machines.)*
5. **You rent computers by the hour from someone else's building.** Need 100 more? They appear in two minutes. Give them back tonight. *(Cloud.)*
6. **Instead of a whole pretend computer, you just ship your program in a lightweight box** that starts in a second instead of a minute. *(Containers.)*
7. **You don't even think about computers.** You hand over a function, and it runs when someone calls it. You pay per call. *(Serverless.)*

Each step made computers **faster to get, cheaper to waste, and easier to throw away and replace.**

That last bit — **easy to throw away** — is what changed architecture forever. When a machine was precious you nursed it back to health. When machines are disposable, you just kill the sick one and start another. **That single change is why we build stateless services behind load balancers.**

---

## The timeline and what each era forced

```mermaid
flowchart LR
    A["Mainframe<br/>1960s-70s"] --> B["Client-server<br/>1980s-90s"]
    B --> C["Own datacenter<br/>1990s-2000s"]
    C --> D["Virtualisation<br/>2000s"]
    D --> E["Cloud / IaaS<br/>2006+"]
    E --> F["Containers<br/>2013+"]
    F --> G["Orchestration<br/>2015+"]
    G --> H["Serverless / edge<br/>2015+"]
```

| Era | Provisioning time | Unit of deployment | Failure model | Architecture it produced |
|---|---|---|---|---|
| Mainframe | Months | The machine | Repair it — it's irreplaceable | Monolithic, batch, time-shared |
| Client-server | Weeks | A server | Repair it | Two/three-tier apps |
| Own datacenter | 6–12 weeks | A rack unit | Repair; keep spares | Fixed capacity, vertical scaling, over-provisioning |
| Virtualisation | Hours | A VM image | Restart the VM | Higher density; VM sprawl |
| Cloud (IaaS) | **Minutes** | An instance | **Replace it** | Autoscaling, stateless services, immutable infrastructure |
| Containers | **Seconds** | An image | Replace it | Microservices become practical |
| Orchestration | Seconds, automatic | A pod/spec | Automatic replacement | Declarative, self-healing |
| Serverless | **Milliseconds** | A function | Not your problem | Event-driven, pay-per-use |

🎯 **The single most important line in that table is "failure model."** When provisioning took six weeks, a dead server was a crisis and you designed to *prevent* failure. When provisioning takes seconds, a dead server is a non-event and you design to *absorb* failure. That inversion is the foundation of everything in Part 3.

---

## Pets vs cattle

The classic metaphor, and it is genuinely useful.

| **Pets** | **Cattle** |
|---|---|
| Named individually (`web-prod-01`, `zeus`) | Numbered (`web-7f3a2b`) |
| Configured by hand, or by accumulated scripts | Built from an image, identically |
| When sick, you nurse it back to health | When sick, you shoot it and start another |
| Unique — nobody knows exactly what's on it | Interchangeable and reproducible |
| Downtime is an incident | Downtime is Tuesday |
| **Snowflake servers**: nobody dares touch them | Any one can be replaced at any moment |

**Everything in modern architecture assumes cattle:**
- Autoscaling requires interchangeable instances.
- Rolling deploys require the ability to kill and replace.
- Health-check-based failover requires disposability.
- Immutable infrastructure requires rebuild-not-patch.

☠️ **Anything that makes an instance special makes it a pet again:** local session state, a local file that matters, a manually applied patch, a hand-edited config, a hard-coded IP that other things depend on. Each one blocks autoscaling and safe deploys.

---

## Virtualisation vs containers

```
┌──────────────────────────┐   ┌──────────────────────────┐
│ App A │ App B │ App C    │   │ App A │ App B │ App C    │
│ Bins  │ Bins  │ Bins     │   │ Bins  │ Bins  │ Bins     │
│ Guest │ Guest │ Guest OS │   ├──────────────────────────┤
│  OS   │  OS   │          │   │   Container runtime      │
├──────────────────────────┤   ├──────────────────────────┤
│      Hypervisor          │   │        Host OS           │
├──────────────────────────┤   ├──────────────────────────┤
│      Hardware            │   │        Hardware          │
└──────────────────────────┘   └──────────────────────────┘
        VMs                          Containers
```

| | VMs | Containers |
|---|---|---|
| Isolation | Strong (separate kernel) | Weaker (shared kernel, namespaces + cgroups) |
| Start time | 30 s – 2 min | 0.1 – 2 s |
| Image size | GBs | MBs |
| Density per host | ~10–50 | ~100–1,000 |
| Overhead | ~5–10% | ~0–2% |
| OS diversity | Any guest OS | Must match the host kernel |
| Security boundary | Suitable for hostile multi-tenancy | Needs extra layers (gVisor, Firecracker, Kata) for hostile tenants |

⚖️ **The trade:** containers give density and speed by *sharing the kernel*, which is exactly what weakens isolation. Cloud providers run untrusted customer containers inside lightweight VMs (Firecracker microVMs) to get both — worth mentioning if a design involves running third-party code.

**Containers matter architecturally because of one property: the image.** A container image bundles the application *and* its dependencies into an immutable artifact that runs identically on a laptop, in CI, and in production. That is what removed "works on my machine" and made deploying 50 services per day feasible.

---

## Orchestration: what Kubernetes actually gives you

Not "containers" — you had those already. Orchestration provides:

| Capability | What it replaces |
|---|---|
| **Declarative desired state** | Imperative deploy scripts |
| **Self-healing** (restart, reschedule) | Someone getting paged |
| **Service discovery + load balancing** | Hand-maintained config ([Service Discovery](../../02-microservices-and-dataflow/01-synchronous-communication/09-service-discovery.md)) |
| **Rolling updates and rollbacks** | Manual, risky deploys |
| **Horizontal autoscaling** | Capacity guesswork |
| **Bin-packing across nodes** | Manual placement, wasted capacity |
| **Config and secret injection** | Baked-in configuration |
| **Health probes gating traffic** | Serving from broken instances |

```yaml
# The declarative model in one snippet
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 5                       # desired state
  strategy:
    rollingUpdate: { maxUnavailable: 1, maxSurge: 2 }
  template:
    spec:
      topologySpreadConstraints:    # spread across AZs
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
      containers:
        - name: api
          image: myapp:1.4.2        # immutable tag, never :latest
          resources:
            requests: { cpu: 200m, memory: 256Mi }   # scheduling
            limits:   { cpu: "1",  memory: 512Mi }   # enforcement
          readinessProbe: { httpGet: { path: /readyz, port: 8080 } }
          livenessProbe:  { httpGet: { path: /livez,  port: 8080 } }
```

⚠️ **Requests vs limits** is the most consequential detail here. `requests` drives scheduling (how much the scheduler reserves); `limits` drives enforcement. Exceeding a **memory** limit gets the container OOM-killed instantly; exceeding a **CPU** limit gets it *throttled*, which silently destroys tail latency and is one of the hardest-to-diagnose performance problems in Kubernetes.

⚖️ **Kubernetes is not free.** It is a distributed system you now operate, with its own failure modes, upgrade cycles, and a steep learning curve. For a small team running three services, a managed platform (Cloud Run, App Runner, Fly, ECS Fargate, Heroku-likes) is usually the better engineering decision. Saying this in an interview is a maturity signal, not a weakness.

---

## Serverless

| Model | Examples | Billing | Cold start |
|---|---|---|---|
| **FaaS** | Lambda, Cloud Functions, Azure Functions | Per invocation + GB-second | 100 ms – 5 s |
| **Container-serverless** | Cloud Run, Fargate, Container Apps | Per vCPU-second while running | 1–5 s |
| **Edge functions** | Cloudflare Workers, Deno Deploy, Lambda@Edge | Per request | ~0 (isolates, not containers) |
| **Serverless data** | DynamoDB on-demand, Aurora Serverless, S3 | Per operation / per GB | N/A |

| ✅ Good fit | ❌ Poor fit |
|---|---|
| Spiky or unpredictable traffic | Steady high traffic (cost crosses over — often 3–5× more than instances) |
| Event-driven glue (S3 → process → DB) | Long-running jobs (execution time limits) |
| Low baseline, occasional bursts | Latency-critical paths sensitive to cold starts |
| Small teams avoiding operations | Workloads needing large memory, GPUs, or special hardware |
| Cron and webhooks | Applications needing persistent connections (WebSocket servers) |

☠️ **Two serverless traps worth naming:**
1. **Cold starts** — mitigated by provisioned concurrency (which reverts you to paying for idle capacity, i.e. servers) or by using isolate-based edge runtimes.
2. **Connection exhaustion** — 1,000 concurrent Lambdas each opening a database connection will destroy a Postgres. You need a proxy (RDS Proxy) or a serverless-native datastore (DynamoDB, HTTP-based drivers).

---

## What this history means for your designs

| Old constraint | Modern reality | Design consequence |
|---|---|---|
| Servers are expensive and slow to get | Elastic and per-second billed | Scale out, don't over-provision |
| Servers are precious | Servers are disposable | Stateless services, externalised state |
| Failure is exceptional | Failure is continuous | Design for absorption, not prevention |
| Deploys are risky events | Deploys are routine and automated | Canary, feature flags, fast rollback |
| Capacity is fixed | Capacity is a dial | Autoscaling, load shedding |
| One big machine | Many small machines | Distributed systems problems become *your* problems |
| Configuration drifts | Immutable images | Rebuild, never patch in place |

🎯 **In an interview:** *"I'd run stateless containers on a managed orchestrator across three AZs, with autoscaling on request rate. The key constraint that makes that work is that any instance can be killed at any moment — so sessions live in Redis, uploads go straight to object storage, and no instance holds anything that can't be rebuilt."* That connects the history to a concrete decision, which is the point of this chapter.

---

## 📋 Chapter checklist

| Skill | Ready? |
|---|---|
| Explain how provisioning time changed the failure model | ☐ |
| Explain pets vs cattle and list five things that create a pet | ☐ |
| Compare VMs and containers on six dimensions | ☐ |
| Explain why the container *image* was the architectural change | ☐ |
| List what orchestration adds beyond containers | ☐ |
| Explain requests vs limits and CPU throttling | ☐ |
| State when serverless is and isn't appropriate | ☐ |
| Name the serverless connection-exhaustion trap | ☐ |

---

**← Previous** [2.2.7 Kafka Exercise](../../02-microservices-and-dataflow/02-asynchronous-communication/07-kafka-exercise.md)
**Next →** [3.1.2 Evolution of a Web App: Stateless vs Stateful](02-stateless-vs-stateful.md)
