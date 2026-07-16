# Enterprise Cloud-Native Architecture Handbook

## Part 6 – Requirements & Capacity Planning

> **Purpose of this chapter**
>
> Before selecting node sizes, HPA limits, KEDA limits, or database SKUs, we must answer one question:
>
> **"What workload are we designing for?"**
>
> Capacity planning converts business expectations into concrete infrastructure sizing.

---

# 6.1 Capacity Planning Philosophy

One of the biggest mistakes teams make is sizing infrastructure based on the current load.

Instead, infrastructure should be sized using:

```text
Expected Peak Load
        +
Growth Buffer
        +
Failure Buffer
```

A common planning formula is:

```text
Required Capacity = Peak Capacity × Growth Factor × Failure Factor
```

**Example**
```text
Current Peak     = 5,000 Requests/sec
Expected Growth   = 2x
Failure Buffer    = 20%

Required Capacity = 5000 × 2 × 1.2 = 12,000 Requests/sec
```

---

# 6.2 Business Assumptions

These assumptions drive all calculations.

| Parameter             | Example |
| --------------------- | ------- |
| Daily Active Users    | 500,000 |
| Peak Concurrent Users | 25,000  |
| Peak Requests/sec     | 6,000   |
| Background Jobs/sec   | 2,500   |
| Average API Response  | 150 ms  |
| Average DB Query      | 20 ms   |
| Average Celery Task   | 2 sec   |

These values should be adjusted based on production traffic or load-testing results.

---

# 6.3 Workload Characteristics

Not every request has the same cost.

Example workload mix:

| Request Type               | Percentage |
| -------------------------- | ---------- |
| Read APIs                  | 70%        |
| Write APIs                 | 20%        |
| Background Task Submission | 10%        |

Implications:

* Most database traffic is read-heavy.
* Read replicas can offload the primary database.
* Write capacity remains focused on the primary instance.

---

# 6.4 API Capacity Planning

## Formula
```text
Required API Pods = Peak Requests/sec / Requests handled per Pod
```

### Example
```text
Peak traffic              = 6,000 Requests/sec
Capacity per Flask pod    = 120 Requests/sec

6000 / 120 = 50 Pods

Safety margin (20%):  50 × 1.2 = 60 Pods
```

**Recommendation:**

| Setting      | Value |
| ------------ | ----- |
| Minimum Pods | 5     |
| Normal Peak  | 50    |
| Maximum Pods | 60    |

---

# 6.5 HPA Planning

Recommended HPA configuration:

| Metric       | Target |
| ------------ | ------ |
| CPU          | 70%    |
| Memory       | 75%    |
| Min Replicas | 5      |
| Max Replicas | 60     |

Scaling flow:
```text
Incoming Traffic → CPU > 70% → HPA → More Flask Pods
→ Cluster Autoscaler (if needed) → Additional AKS Nodes
```

---

# 6.6 Background Worker Capacity

## Formula
```text
Workers Required = Incoming Jobs/sec / Jobs processed/sec by one Worker
```

### Example
```text
Incoming jobs                 = 2,500 Jobs/sec
Jobs processed per Worker     = 25 Jobs/sec

2500 / 25 = 100 Workers

Safety margin (25%):  100 × 1.25 = 125 Workers
```

Recommended KEDA limits:

| Setting     | Value |
| ----------- | ----- |
| Min Workers | 2     |
| Max Workers | 125   |

---

# 6.7 RabbitMQ Queue Planning

Queues should absorb temporary spikes.

```text
Traffic spike (10,000 Jobs) → RabbitMQ Queue → Workers process gradually
```

Queue depth formula:
```text
Maximum Queue = Maximum Processing Delay × Incoming Jobs/sec
```

### Example
```text
Maximum acceptable delay   = 30 seconds
Incoming jobs              = 2,000/sec

Queue size = 30 × 2000 = 60,000 messages
```

This becomes an operational threshold for alerts.

---

# 6.8 Database Capacity Planning

The database is usually the first bottleneck in a scalable architecture. Everything else must be designed around it.

## Step 1 — Maximum Database Connections

```text
Max Connections = 1,200
```

Reserve:

| Purpose              | Connections |
| -------------------- | ----------- |
| DBA/Admin            | 40          |
| Monitoring           | 30          |
| Backup & Maintenance | 30          |
| **Reserved Total**   | **100**     |

```text
Application budget = 1200 - 100 = 1,100 Connections
```

## Step 2 — API Pool

```text
Maximum API Pods = 60
Pool size per pod = 10

Connections = Pods × Pool Size = 60 × 10 = 600 Connections
```

## Step 3 — Celery Pool *(corrected)*

The connection budget must be validated against KEDA's actual configured maximum (125 workers, per 6.6), not a lower pre-margin figure.

Remaining budget after API pool: `1,100 - 600 = 500 connections`.

To stay within budget at the full 125-worker maximum, the per-worker pool size must be reduced:
```text
Maximum Workers      = 125
Pool size per worker = 4   (reduced from 5 to fit budget)
Connections = 125 × 4 = 500 Connections
```

> **Why reduce pool size instead of lowering the worker count?** The 125-worker maximum is referenced consistently elsewhere in the handbook (Part 5's KEDA boundaries). Reducing the per-worker pool size instead keeps that number consistent while still respecting the database budget.

## Step 4 — Validation

```text
Total application connections = 600 (API) + 500 (Workers) = 1,100
```

This now matches the planned budget **even when both HPA and KEDA are simultaneously at their configured maximum** — which is the scenario that actually matters, since it's the one guaranteed to eventually occur under sustained peak load.

## Capacity Formula
```text
(API Pods × API Pool) + (Workers × Worker Pool) + Reserved ≤ Database Maximum Connections
```

This should always be validated **using each component's configured maximum**, not a lower current or "typical" value — otherwise the check gives a false sense of safety.

---

# 6.9 Read Replica Planning

```text
Reads  = 80%
Writes = 20%
```

```text
                Primary
                  │
      ┌───────────┼───────────┐
 Replica1     Replica2     Replica3
```

Guidelines:

* Writes always go to the primary.
* Read replicas handle reporting and read-heavy APIs.
* Critical read-after-write operations should use the primary to avoid replica lag.

---

# 6.10 AKS Node Planning *(rewritten)*

The original approach to this section treated node sizing as a single step: divide total pod count by a flat "30 pods per node" figure. That number isn't arbitrary, but it isn't a resource calculation either — and using one blended figure across every workload type hides a large amount of real infrastructure cost. This section replaces that approach with the actual two-step process.

## 6.10.1 Where "pods per node" limits actually come from

There are **two independent ceilings** on how many pods fit on a node, and the real number is whichever is lower:

**A. Networking ceiling (fixed by AKS network plugin choice)**

| Network Plugin | Max Pods / Node | Why |
| --- | --- | --- |
| Azure CNI (classic) | ~30 (default; configurable up to 250) | Each pod consumes a routable IP directly from the VNet subnet — this is the source of the "30" figure used in the original section |
| Kubenet | 110 (default) | Pods use a separate, non-VNet-routable IP range |
| Azure CNI Overlay | Up to 250 | Pods get overlay IPs, not VNet IPs — removes the IP-exhaustion constraint |

**B. Resource ceiling (driven by the actual VM size and each pod's CPU/memory *requests*)**

```text
CPU-bound density    = Node Allocatable CPU / Pod CPU Request
Memory-bound density = Node Allocatable Memory / Pod Memory Request

Actual Pods per Node = min(CPU-bound density, Memory-bound density, Network Ceiling)
```

"Allocatable" is not the VM's full advertised CPU/memory — AKS reserves a portion of every node for the OS and kubelet system daemons before any pod can be scheduled. As a planning rule of thumb, budgeting for **~10% of node CPU and memory as system-reserved** is a reasonable estimate for mid-sized general-purpose VMs (exact reservation depends on VM size and should be confirmed against current AKS documentation before finalizing a sizing decision).

## 6.10.2 Choosing a VM SKU per node pool

Because each workload in this architecture has a different resource shape, each node pool should be sized independently rather than sharing one VM size:

| Node Pool | Workload shape | Suggested VM family |
| --- | --- | --- |
| API | Moderate CPU, light memory (0.5 CPU / 0.5 GiB request) | General purpose (e.g. Dsv5) |
| Worker | Higher CPU ceiling, moderate memory | General purpose / compute-optimized (e.g. Fsv2) |
| Messaging (RabbitMQ) | Memory-heavy, needs stable low-latency neighbors | Memory-optimized (e.g. Esv5) |
| System/Platform | Small, steady, always-on (ingress, monitoring, Beat) | Small general purpose (e.g. Dsv5, smaller size) |

## 6.10.3 Worked example — API node pool

Using the pod requests from Section 6.12 (500m CPU / 512 MiB per Flask pod) on a 4 vCPU / 16 GiB VM, with ~10% reserved for system overhead:


```text
Pod request: 500m CPU / 512 MiB   (from 6.12)
VM: 4 vCPU / 16 GiB → Allocatable ≈ 3.6 vCPU, 14.4 GiB

CPU-bound density    = 3.6 / 0.5   ≈ 7 pods
Memory-bound density = 14.4 / 0.5  ≈ 28 pods
Network ceiling       = 30 pods

Binding constraint = min(7, 28, 30) = 7 pods/node   (CPU-bound)
```
```text
Max API Pods = 60
Nodes required = ceil(60 / 7) = 9 nodes
```

This is markedly different from what a flat "30 pods/node" assumption would imply (60/30 = 2 nodes) — the resource ceiling, not the networking ceiling, is what actually binds here.

## 6.10.4 Worked example — Worker node pool

Using the Celery worker request (500m CPU / 1 GiB per pod) on an 8 vCPU / 16 GiB compute-optimized VM:

```text
Pod request: 500m CPU / 1 GiB   (from 6.12)
VM: 8 vCPU / 16 GiB
Allocatable ≈ 7.2 vCPU, 14.4 GiB

CPU-bound density    = 7.2 / 0.5 ≈ 14 pods
Memory-bound density = 14.4 / 1  ≈ 14 pods
Binding constraint = min(14, 14, 30) = 14 pods/node

Max Worker Pods = 125 → 
Nodes = ceil(125 / 14) = 9 nodes
```

## 6.10.5 Worked example — Messaging node pool

RabbitMQ pods request 2 CPU / 4 GiB each (from 6.12), and there are 3 fixed replicas. On a 4 vCPU / 32 GiB memory-optimized VM:

```text
Pod request: 2 CPU / 4 GiB   (from 6.12)
VM: 4 vCPU / 32 GiB → 
Allocatable ≈ 3.6 vCPU, 28.8 GiB

CPU-bound density    = 3.6 / 2  ≈ 1 pod
Memory-bound density = 28.8 / 4 ≈ 7 pods
Binding constraint = min(1, 7) = 1 pod/node

RabbitMQ Pods = 3 → Nodes = 3 nodes
```
One replica per node is desirable here, not just an artifact — it isolates each quorum member from node-level failure of its peers.

## 6.10.6 System/Platform node pool — assumed values *(gap filled)*

The original handbook did not document CPU/memory requests for Ingress, Monitoring, or System pods — only Flask API, Celery Worker, RabbitMQ, and Celery Beat were specified in 6.12. The values below are **reasonable assumptions for illustration, not original handbook data**, and should be replaced with real profiled numbers before use in an actual capacity plan.

| Component | Pods | Assumed CPU Request | Assumed Memory Request | Basis for assumption |
| --- | --- | --- | --- | --- |
| Ingress (NGINX) | 2 | 200m | 256 MiB | Typical published sizing for NGINX ingress controller at moderate traffic |
| Monitoring (Prometheus/Grafana/Loki + exporters, blended average) | 10 | 300m | 512 MiB | Prometheus is memory-heavier than this average; treat as a blended estimate across a mixed monitoring stack, not any single component |
| System (cluster add-ons: CoreDNS, metrics-server, CSI drivers, etc.) | 15 | 100m | 128 MiB | Typical lightweight footprint for small cluster add-on pods |
| Celery Beat | 1 | 250m | 256 MiB | *(this one is real — documented in 6.12, not assumed)* |

Because this pool is a **heterogeneous mix** of pod shapes rather than one repeated pod type, node count here is best computed from total resource demand rather than per-pod density:

```text
Total CPU requested:
  Ingress:    2 × 200m  = 400m
  Monitoring: 10 × 300m = 3,000m
  System:     15 × 100m = 1,500m
  Beat:       1 × 250m  = 250m
  Sum = 5,150m ≈ 5.15 vCPU

Total Memory requested:
  Ingress:    2 × 256 MiB  = 512 MiB
  Monitoring: 10 × 512 MiB = 5,120 MiB
  System:     15 × 128 MiB = 1,920 MiB
  Beat:       1 × 256 MiB  = 256 MiB
  Sum = 7,808 MiB ≈ 7.6 GiB

VM: 4 vCPU / 16 GiB → Allocatable ≈ 3.6 vCPU, 14.4 GiB per node

Nodes by CPU    = ceil(5.15 / 3.6) = 2 nodes
Nodes by Memory = ceil(7.6 / 14.4) = 1 node
Nodes by pod count (28 pods, network ceiling 30/node) = 1 node

Binding constraint = 2 nodes (CPU-bound)
```

## 6.10.7 Combined node count — per pool

| Node Pool | Pods at Max | Density/Node | Nodes Required |
| --- | --- | --- | --- |
| API | 60 | 7 (CPU-bound) | 9 |
| Worker | 125 | 14 (CPU/Memory tied) | 9 |
| Messaging | 3 | 1 (CPU-bound) | 3 |
| System/Platform | 28 | ~14 (blended — heterogeneous mix, CPU-bound by sum) | 2 |
| **Total** | **216** | | **~23 nodes** |

This confirms the earlier estimate: computing per-pool from actual (or reasonably assumed) resource shape gives **~23 nodes at full scale**, not the 8 nodes the original blended "30 pods/node" calculation produced. The API and Worker pools are CPU-request-bound, not IP-count-bound, at realistic VM sizes — that's the main driver of the difference.

---

# 6.11 Autoscaling Limits

Recommended limits:

| Component      | Min | Max |
| -------------- | --- | --- |
| Flask API      | 5   | 60  |
| Celery Workers | 2   | 125 |
| RabbitMQ       | 3   | 3   |
| Celery Beat    | 1   | 1   |
| Cluster Nodes  | 3   | ~23 *(revised — see 6.10.7; original said 10)* |

These values should be revisited after load testing.

---

# 6.12 Resource Requests and Limits

**Documented in original handbook:**

| Component     | CPU Request | CPU Limit | Memory Request | Memory Limit |
| ------------- | ----------- | --------- | -------------- | ------------ |
| Flask API     | 500m        | 1 CPU     | 512 MiB        | 1 GiB        |
| Celery Worker | 500m        | 2 CPU     | 1 GiB          | 2 GiB        |
| RabbitMQ      | 2 CPU       | 4 CPU     | 4 GiB          | 8 GiB        |
| Celery Beat   | 250m        | 500m      | 256 MiB        | 512 MiB      |

**Assumed in this revision (not in original handbook — see 6.10.6):**

| Component  | CPU Request (assumed) | Memory Request (assumed) |
| ---------- | ---------------------- | -------------------------- |
| Ingress    | 200m                    | 256 MiB                    |
| Monitoring | 300m                    | 512 MiB                    |
| System     | 100m                    | 128 MiB                    |

All values should be tuned based on real profiling rather than guessed — this applies doubly to the assumed rows above.

---

# 6.13 Capacity Validation Checklist

Before production, verify:

* ✔ Peak RPS is known.
* ✔ Peak queue throughput is known.
* ✔ Average and P95 API latency are measured.
* ✔ Average Celery task duration is measured.
* ✔ Database `max_connections` is documented.
* ✔ Connection pools are sized to stay within the database budget **at each component's configured maximum, not just current load**.
* ✔ HPA maximum replicas respect database capacity.
* ✔ KEDA maximum replicas respect database capacity.
* ✔ AKS node limits can accommodate the maximum pod count **using per-pool resource-based density, not a single blended pods-per-node figure**.
* ✔ VM SKU per node pool is chosen based on that pool's actual CPU/memory request shape.
* ☐ **Ingress, Monitoring, and System pod resource requests are profiled with real numbers** (currently assumed — see 6.10.6/6.12).
* ✔ Load tests validate all assumptions.

---

# 6.14 Chapter Summary

Capacity planning is about ensuring every layer scales together without overwhelming another.

```text
Business Demand
        │
        ▼
Expected RPS & Job Rate
        │
        ▼
API Pods (HPA)
        │
        ▼
Worker Pods (KEDA)
        │
        ▼
Database Connections
        │
        ▼
AKS Nodes (Cluster Autoscaler)
```

The database often defines the practical upper limit for the entire system. By calculating connection budgets first — **against configured maximums, not current values** — and then deriving HPA and KEDA limits from that budget, the platform can scale predictably without exhausting shared resources. Node counts should then be derived per node pool, from each workload's actual resource shape, not a single blended pods-per-node assumption.

| Layer          | Primary Metric | Planning Formula               |
| -------------- | -------------- | ------------------------------ |
| Flask API      | Requests/sec   | RPS ÷ RPS per Pod              |
| Celery Workers | Jobs/sec       | Jobs ÷ Worker Throughput       |
| RabbitMQ       | Messages/sec   | Messages ÷ Consumer Throughput |
| MySQL          | Connections    | Pods × Pool Size (at max scale)|
| AKS            | Pod Density    | min(CPU-bound, Memory-bound, Network ceiling) density, per node pool |

---

# Next Chapter

➡ **Part 7 – Database Design**