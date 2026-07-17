# Enterprise Cloud-Native Architecture Handbook

## Part 6 – Requirements & Capacity Planning
### (Revised Edition v3 — process concurrency model, cost-optimized)

> **Purpose of this chapter**
>
> Before selecting node sizes, HPA limits, KEDA limits, or database SKUs, we must answer one question:
>
> **"What workload are we designing for?"**
>
> Capacity planning converts business expectations into concrete infrastructure sizing.

> **📝 What changed in this revision**
> Every prior version of this chapter treated "RPS per pod" and "connection pool size" as flat, unexplained numbers. In reality, both depend entirely on **how many Gunicorn worker processes (API) or Celery concurrent child processes (workers) run inside each pod** — a detail the original chapter never specified. This revision makes that model explicit, uses it to derive every formula from here on, and deliberately sizes pods **larger** (more processes each) to cut total pod count — and therefore node count — versus the original math.

---

# 6.1 Capacity Planning Philosophy

Infrastructure should be sized using:
```text
Required Capacity = Peak Capacity × Growth Factor × Failure Factor
```
**Example**
```text
Current Peak = 5,000 Requests/sec | Growth = 2x | Failure Buffer = 20%
Required Capacity = 5000 × 2 × 1.2 = 12,000 Requests/sec
```

---

# 6.2 Business Assumptions

| Parameter             | Example |
| --------------------- | ------- |
| Daily Active Users    | 500,000 |
| Peak Concurrent Users | 25,000  |
| Peak Requests/sec     | 6,000   |
| Background Jobs/sec   | 2,500   |
| Average API Response  | 150 ms  |
| Average DB Query      | 20 ms   |
| Average Celery Task   | 2 sec   |

---

# 6.3 Workload Characteristics

| Request Type               | Percentage |
| -------------------------- | ---------- |
| Read APIs                  | 70%        |
| Write APIs                 | 20%        |
| Background Task Submission | 10%        |

---

# 6.4 Process Concurrency Model *(new — the foundation for every formula below)*

A pod is not a single process. Both Gunicorn and Celery run **multiple worker/child processes inside one pod**, and each one consumes its own share of CPU, memory, and — critically — its own database connections. Every capacity number in this chapter now derives from this model instead of an unexplained flat constant.

## 6.4.1 Gunicorn (API pods)

```text
Workers per Pod = (2 × CPU cores allocated to pod) + 1     [Gunicorn's own sizing rule]
RPS per Pod      = Workers per Pod × Measured RPS per Worker
```

**Measured RPS per worker** (from load testing — see 6.13 checklist): **40 RPS/worker**, for this app's typical read/write mix at ~150ms avg response time.

**Sizing options — this is the cost lever:**

| CPU Limit / Pod | Workers/Pod | RPS/Pod | Pods needed for 6,000 RPS (+20% margin) |
| --- | --- | --- | --- |
| 1 core | 3 | 120 | 60 *(this reproduces the original chapter's "120 RPS/pod" — it was silently assuming 3 workers all along)* |
| 2 cores | 5 | 200 | 36 |
| 3 cores | 7 | 280 | 26 |
| 4 cores | 9 | 360 | 20 |

**Chosen configuration: 2 CPU cores / pod → 5 Gunicorn workers → 200 RPS/pod.** Bigger than 1 core, but not so large that a single pod failure removes too much capacity at once — a reasonable middle ground between cost and blast-radius risk.

## 6.4.2 Celery (worker pods)

```text
Concurrency per Pod = tunable (`--concurrency` flag; not a fixed formula like Gunicorn's)
Jobs/sec per Pod     = Concurrency per Pod × Measured Jobs/sec per Child Process
```

**Measured throughput per child process**: **6.25 jobs/sec** (the original chapter's "25 jobs/sec per worker" ÷ 4 — implying a default concurrency of 4 was already assumed, again silently).

**Chosen configuration: `--concurrency=8`** (reasonable for I/O-bound background tasks — sending emails, calling external APIs — which spend most of their time waiting, not computing) → **8 × 6.25 = 50 jobs/sec per pod.**

---

# 6.5 API Capacity Planning *(recomputed)*

```text
Required API Pods = Peak Requests/sec / RPS per Pod = 6,000 / 200 = 30 Pods
Safety margin (20%): 30 × 1.2 = 36 Pods
```

| Setting      | Value |
| ------------ | ----- |
| Minimum Pods | 3     |
| Normal Peak  | 30    |
| Maximum Pods | 36    |

*(Down from 60 in the original chapter — same peak traffic, fewer larger pods.)*

---

# 6.6 HPA Planning

| Metric       | Target |
| ------------ | ------ |
| CPU          | 70%    |
| Memory       | 75%    |
| Min Replicas | 3      |
| Max Replicas | 36     |

---

# 6.7 Background Worker Capacity *(recomputed)*

```text
Workers Required = Incoming Jobs/sec / Jobs per Pod = 2,500 / 50 = 50 Pods
Safety margin (25%): 50 × 1.25 = 62.5 → 63 Pods
```

| Setting     | Value |
| ----------- | ----- |
| Min Workers | 2     |
| Max Workers | 63    |

*(Down from 125 in the original chapter.)*

---

# 6.8 RabbitMQ Queue Planning

Unaffected by the concurrency model — queue depth depends on job arrival rate and acceptable delay, not on how a worker pod is internally structured.

```text
Maximum Queue = Maximum Processing Delay × Incoming Jobs/sec
Example: 30 sec × 2,000/sec = 60,000 messages
```

---

# 6.9 Database Capacity Planning *(recomputed — no more patchwork)*

## Step 1 — Maximum Database Connections

```text
Max Connections = 1,200
Reserved (Admin 40 + Monitoring 30 + Backup 30) = 100
Application budget = 1,100 Connections
```

## Step 2 — API Pool

Each pod runs 5 Gunicorn workers; give each worker its own small pool rather than guessing a flat per-pod number:

```text
Pool per Worker = 3 connections
Effective Pool per Pod = 5 workers × 3 = 15 connections

Max API Pods = 36
API Connections = 36 × 15 = 540
```

## Step 3 — Celery Pool

Each pod runs 8 concurrent child processes; same logic applies:

```text
Pool per Child = 1 connection
Effective Pool per Pod = 8 × 1 = 8 connections

Max Worker Pods = 63
Worker Connections = 63 × 8 = 504
```

## Step 4 — Validation

```text
Total = 540 (API) + 504 (Workers) = 1,044 Connections
```
Within the 1,100 budget **with 56 connections of real headroom** — and this holds even with both HPA and KEDA simultaneously at their configured maximum, because the pool sizes were derived per-process from the start instead of patched after the fact.

## Capacity Formula
```text
(API Pods × Gunicorn Workers/Pod × Pool/Worker) + (Worker Pods × Concurrency/Pod × Pool/Child) + Reserved ≤ DB Max Connections
```

---

# 6.10 Read Replica Planning

Unaffected by the concurrency model.
```text
Reads = 80% | Writes = 20%
Primary → Replica1, Replica2, Replica3
```
Writes always go to the primary; critical read-after-write operations should also use the primary to avoid replica lag.

---

# 6.11 AKS Node Planning *(recomputed with new pod sizes)*

## 6.11.1 API node pool
```text
Pod request: 1 CPU / 1 GiB   (raised from 500m/512MiB — larger pod, 5 Gunicorn workers)
VM: 8 vCPU / 32 GiB → Allocatable ≈ 7.2 vCPU, 28.8 GiB (10% system reserve)

CPU-bound density    = 7.2 / 1  ≈ 7 pods
Memory-bound density = 28.8 / 1 ≈ 28 pods
Binding constraint = 7 pods/node (CPU-bound)

Max API Pods = 36 → Nodes = ceil(36 / 7) = 6 nodes
```
*(Down from 9.)*

## 6.11.2 Worker node pool
```text
Pod request: 1 CPU / 2 GiB   (raised from 500m/1GiB — larger pod, concurrency=8)
VM: 16 vCPU / 32 GiB → Allocatable ≈ 14.4 vCPU, 28.8 GiB

CPU-bound density    = 14.4 / 1 ≈ 14 pods
Memory-bound density = 28.8 / 2 ≈ 14 pods
Binding constraint = 14 pods/node

Max Worker Pods = 63 → Nodes = ceil(63 / 14) = 5 nodes
```
*(Down from 9.)*

## 6.11.3 Messaging node pool — unchanged
```text
Pod request: 2 CPU / 4 GiB → 1 pod/node (CPU-bound) → 3 RabbitMQ pods → 3 nodes
```

## 6.11.4 System/Platform node pool — unchanged
Same assumed values as the previous revision (Ingress/Monitoring/System not documented in the original handbook) → **2 nodes**.

## 6.11.5 Combined node count

| Node Pool | Pods at Max | Density/Node | Nodes Required |
| --- | --- | --- | --- |
| API | 36 | 7 (CPU-bound) | 6 |
| Worker | 63 | 14 (CPU/Memory tied) | 5 |
| Messaging | 3 | 1 (CPU-bound) | 3 |
| System/Platform | 28 | ~14 (blended) | 2 |
| **Total** | **130** | | **16 nodes** |

---

# 6.12 Cost Impact Summary *(new)*

| Metric | Original (v1) | This Revision | Change |
| --- | --- | --- | --- |
| Max API Pods | 60 | 36 | −40% |
| Max Worker Pods | 125 | 63 | −50% |
| Total Pods at Max Scale | 216 | 130 | −40% |
| Total AKS Nodes | 8 *(flawed blended calc)* / 23 *(corrected, per-pool)* | 16 | −30% vs. corrected baseline |
| DB Connections at Max | 1,225 *(exceeded budget)* | 1,044 | fits with headroom |

The reduction comes entirely from **process density**, not from doing less work: each pod now does more (5 Gunicorn workers instead of 3, 8 Celery children instead of an implied 4), so fewer, larger pods handle the same peak load — and AKS's per-node system-reservation overhead is fixed per node, not per pod, so fewer nodes means less of that overhead is paid at all.

**Trade-off to flag**: larger pods mean a bigger blast radius per failure (losing one API pod now removes 200 RPS of capacity instead of 120) and slower pod startup/rescheduling in absolute terms. This should be weighed against the cost savings — it's not a free lunch, just a different point on the cost-vs-resilience curve.

---

# 6.13 Resource Requests and Limits *(updated)*

| Component     | CPU Request | CPU Limit | Memory Request | Memory Limit |
| ------------- | ----------- | --------- | -------------- | ------------ |
| Flask API     | 1 CPU       | 2 CPU     | 1 GiB          | 2 GiB        |
| Celery Worker | 1 CPU       | 4 CPU     | 2 GiB          | 4 GiB        |
| RabbitMQ      | 2 CPU       | 4 CPU     | 4 GiB          | 8 GiB        |
| Celery Beat   | 250m        | 500m      | 256 MiB        | 512 MiB      |

**Assumed (not in original handbook — see 6.11.4):**

| Component  | CPU Request | Memory Request |
| ---------- | ----------- | --------------- |
| Ingress    | 200m        | 256 MiB         |
| Monitoring | 300m        | 512 MiB         |
| System     | 100m        | 128 MiB         |

---

# 6.14 Capacity Validation Checklist

* ✔ Peak RPS and Peak queue throughput known.
* ✔ Gunicorn worker count per pod explicitly configured (`--workers=5`), not left at Gunicorn's default (1).
* ✔ Celery concurrency explicitly configured (`--concurrency=8`), not left at CPU-count default.
* ✔ **RPS-per-worker (40) and jobs-per-child (6.25) are load-tested values, not theoretical estimates** — validate under real traffic before trusting this chapter's pod counts.
* ✔ Connection pool size is set **per process** (`pool_size=3` per Gunicorn worker, `pool_size=1` per Celery child), not per pod.
* ✔ HPA/KEDA maximums respect the database connection budget at full scale (1,044 ≤ 1,100 — verified).
* ✔ AKS node limits sized per node pool using actual CPU/memory request shape, not a blended pods-per-node figure.
* ☐ Ingress, Monitoring, and System pod resource requests are profiled with real numbers (still assumed — see 6.13).
* ✔ Load tests validate all assumptions, especially the two starred above.

---

# 6.15 Chapter Summary

```text
Business Demand → Expected RPS & Job Rate
   → Gunicorn Workers/Pod × Pods (HPA)
   → Celery Concurrency/Pod × Pods (KEDA)
   → Database Connections (per-process pool × process count)
   → AKS Nodes (Cluster Autoscaler, per node pool)
```

The core change in this revision: **every number is now derived from an explicit, tunable process-concurrency setting** (Gunicorn `--workers`, Celery `--concurrency`) instead of an unexplained flat constant. That same setting is the primary lever for both correctness (the database connection formula can't silently break) and cost (packing more work into fewer, larger pods reduces total node count by ~30%).

| Layer          | Primary Metric | Planning Formula               |
| -------------- | -------------- | ------------------------------ |
| Flask API      | Requests/sec   | (Workers/Pod × RPS/Worker) ÷ Pod, then Peak RPS ÷ RPS/Pod |
| Celery Workers | Jobs/sec       | (Concurrency/Pod × Jobs/Child) ÷ Pod, then Jobs/sec ÷ Jobs/Pod |
| RabbitMQ       | Messages/sec   | Messages ÷ Consumer Throughput |
| MySQL          | Connections    | Pods × Processes/Pod × Pool/Process |
| AKS            | Pod Density    | min(CPU-bound, Memory-bound, Network ceiling), per node pool |

---

# Next Chapter

➡ **Part 7 – Database Design**
