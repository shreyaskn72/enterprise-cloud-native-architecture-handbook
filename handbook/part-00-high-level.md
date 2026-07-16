# Cloud-Native Architecture — high level

## 1. System in one line
A Flask API + Celery background workers, connected via RabbitMQ, backed by Azure MySQL, all running on AKS with autoscaling at every layer. Sync work → hits DB directly. Async/slow work → queued and processed later.

| Component | Choice |
|---|---|
| API | Flask + Gunicorn |
| Broker | RabbitMQ (3-node quorum) |
| Background Jobs | Celery + Celery Beat (1 replica) |
| Database | Azure MySQL (1 primary + read replicas) |
| Autoscaling | HPA (API) + KEDA (workers) + Cluster Autoscaler (nodes) |
| Monitoring | Prometheus + Grafana + Loki + OpenTelemetry |
| Secrets | Azure Key Vault |
| IaC | Helm (Terraform/Crossplane — planned) |

## 2. Request Flow
Sync path answers immediately; async path replies "got it" and finishes the job in the background — keeps the API fast.
```
User → Front Door/App Gateway → NGINX Ingress → Flask API (stateless)
   ├─ Sync  → MySQL Primary → Read Replicas (replication)
   └─ Async → RabbitMQ → Celery Workers (KEDA) → MySQL Primary
Celery Beat (scheduler) → RabbitMQ
```

## 3. Autoscaling — who watches what
HPA/KEDA add pods; Cluster Autoscaler adds nodes when pods have nowhere to run. They're a chain, not independent.

| Autoscaler | Watches | Scales |
|---|---|---|
| HPA | CPU/Memory | API pods |
| KEDA | RabbitMQ queue length | Worker pods |
| Cluster Autoscaler | Pending pods | AKS nodes |

**Formulas:**
```
HPA:   Desired Replicas = Current Replicas × (Current Metric / Target Metric)
KEDA:  Workers Needed = Queue Length / Messages per Worker
```
Targets: CPU 70%, Memory 75%. Stabilization/cooldown: 300s (prevents flapping).

## 4. Capacity Planning — size everything with math, not guesses
Before you set any autoscaling limit, calculate how much you actually need — these five formulas cover every layer of the stack.
```
API Capacity:        Pods Required = Peak RPS / RPS per Pod
Worker Capacity:     Workers = Incoming Jobs/sec / Jobs processed/sec
DB Connections:      Total Connections = (API Pods × Pool Size) + (Workers × Pool Size) + Admin + Monitoring
RabbitMQ Throughput: Consumer Count = Messages/sec / Worker Throughput
AKS Nodes:           Nodes Required = Pods / Pods per Node
```

## 5. The database is the real ceiling
No matter how many pods you spin up, the database's max connection limit is the hard stop — always size autoscaling limits from this, not arbitrary numbers.
```
(API Pods × Pool Size) + (Worker Pods × Pool Size) + Reserved ≤ DB Max Connections
```
Min/Max example: Flask 5–60 · Workers 2–125 · Nodes 3–12

## 6. Failure & Recovery
Kubernetes and Azure MySQL are both designed to self-heal — the pattern is always: **detect → isolate → recover**.

| Failure | Recovery |
|---|---|
| API pod crash | Restart → health probe validates |
| RabbitMQ node down | Quorum re-elects leader, no message loss |
| Primary DB down | Auto-failover → replica promoted |
| AKS node down | Pods evicted → rescheduled elsewhere |

**RTO** = how long you can be down. **RPO** = how much data you can afford to lose. **PITR** = restore DB to any exact past moment.

## 7. Observability — 3 pillars
Metrics tell you *something's wrong*, logs tell you *what happened*, traces tell you *where the time went*.

| Pillar | Answers | Tool |
|---|---|---|
| Metrics | How much/how many? | Prometheus + Grafana |
| Logs | What exactly happened? | Loki |
| Traces | Where was the request slow? | OpenTelemetry |

## 8. Security — defense in depth
Layer multiple protections so one failure doesn't expose everything.
- **Access**: Azure RBAC + K8s RBAC
- **Secrets**: Key Vault (never hardcoded)
- **Network**: TLS + Network Policies (pod-to-pod rules)
- **Containers**: image scanning before deploy

## 9. Advanced patterns — name + one-liner
- **Circuit Breaker** — stop calling a failing service temporarily, like a fuse
- **Saga Pattern** — multi-service transactions done as reversible steps
- **CQRS** — separate models for reading vs. writing data
- **Rate Limiting** — cap requests per user/client
- **Cache Coherency** — keeping multiple caches in sync

## 10. Golden rules to remember
1. Autoscaling max limits derive from **DB capacity**, not guesses.
2. Stateless pods + IaC + Observability-first = production-ready foundation.
3. Single Celery Beat replica and Helm-only IaC are known gaps — fine for base design, flagged for hardening later.
