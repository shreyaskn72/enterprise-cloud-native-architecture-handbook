# Enterprise Cloud-Native Architecture Handbook

# Part 5 – Scaling Strategy (HPA, KEDA & Cluster Autoscaler)

> **Objective**
>
> Autoscaling is the heart of a cloud-native architecture. This chapter explains how **Horizontal Pod Autoscaler (HPA)**, **KEDA**, and the **Cluster Autoscaler** work together to provide elastic scaling while protecting shared resources such as the database and message broker. The goal is not simply to scale faster, but to scale **predictably, efficiently, and safely**.

---

# 5.1 Scaling Philosophy

A common misconception is:

> "Scale everything as much as possible."

Instead, the guiding principle should be:

> **Scale only as much as downstream systems can safely handle.**

The application must be viewed as a connected system.

```text
Users
   │
   ▼
Flask API
   │
RabbitMQ
   │
Celery Workers
   │
Azure MySQL
```

If one layer scales without considering the next, the bottleneck simply moves downstream.

---

# 5.2 The Three Layers of Autoscaling

Each autoscaler has a different responsibility.

| Component          | Trigger                       | Purpose         |
| ------------------ | ----------------------------- | --------------- |
| HPA                | CPU / Memory / Custom Metrics | Scale API pods  |
| KEDA               | Queue length / Events         | Scale workers   |
| Cluster Autoscaler | Pending pods                  | Scale AKS nodes |

Think of them as three cooperating systems.

```text
Traffic
   │
   ▼
HPA
   │
More Pods
   │
No Node Capacity
   │
Cluster Autoscaler
```

Meanwhile,

```text
Queue Growth
      │
      ▼
KEDA
      │
More Workers
      │
Need More Nodes
      │
Cluster Autoscaler
```

---

# 5.3 Request Scaling Flow

## Step 1

Traffic increases.

```text
1,000 RPS

↓

6,000 RPS
```

---

## Step 2

API CPU increases.

```text
CPU

35%

↓

85%
```

---

## Step 3

HPA calculates desired replicas.

Formula

```
Desired Replicas

=

Current Replicas

×

(Current Metric / Target Metric)
```

Example

```
Current Pods = 10

CPU = 84%

Target = 70%

Desired

=

10 × (84/70)

=

12 Pods
```

HPA creates 2 new pods.

---

## Step 4

If no node has enough resources,

Pods become

```text
Pending
```

---

## Step 5

Cluster Autoscaler detects pending pods.

```text
Pending Pods

↓

Add Node

↓

Pods Scheduled
```

---

# 5.4 Worker Scaling Flow

Suppose:

```text
Queue Length

=

100
```

Workers:

```text
2
```

Suddenly,

```text
Queue

↓

25,000 Messages
```

KEDA detects the queue increase.

Example formula

```
Workers

=

Queue Length

/

Messages handled per Worker
```

Suppose

Each worker handles

```
100 Messages
```

Calculation

```
25000 / 100

=

250 Workers
```

KEDA requests additional worker pods.

---

# 5.5 Cluster Autoscaler Flow

Suppose

Worker Pool

has

3 nodes.

Maximum pods

```
90
```

KEDA creates

```
200 Worker Pods
```

110 remain pending.

Cluster Autoscaler:

```text
Pending Pods

↓

Add Node

↓

Scheduler

↓

Pods Running
```

---

# 5.6 Complete Scaling Sequence

```text
                  Traffic Spike
                        │
                        ▼
                 Flask API CPU ↑
                        │
                        ▼
                  HPA Adds Pods
                        │
                        ▼
             Enough Node Capacity?
                  │            │
                Yes            No
                 │              │
                 ▼              ▼
         Pods Scheduled   Pending Pods
                                │
                                ▼
                     Cluster Autoscaler
                                │
                                ▼
                          New AKS Nodes
                                │
                                ▼
                         Pods Scheduled
```

Background processing follows a parallel path:

```text
Queue Length ↑
      │
      ▼
    KEDA
      │
      ▼
More Worker Pods
      │
      ▼
Pending?
      │
      ▼
Cluster Autoscaler
```

---

# 5.7 The Database Bottleneck

Autoscaling is meaningless if the database cannot keep up.

Example:

| Component | Max Pods | Pool Size |
| --------- | -------- | --------- |
| Flask     | 60       | 10        |
| Workers   | 125      | 5         |

Connections

```
60 × 10

=

600
```

Workers

```
125 × 5

=

625
```

Total

```
1225
```

Suppose MySQL supports

```
1200
```

The architecture fails.

Instead

Always verify

```
(API Pods × API Pool)

+

(Worker Pods × Worker Pool)

+

Reserved

≤

Database Maximum Connections
```

This is one of the most important formulas in the entire architecture.

---

# 5.8 Choosing HPA Metrics

CPU alone is not always sufficient.

Recommended metrics:

| Metric        | Suitable?               |
| ------------- | ----------------------- |
| CPU           | ✅                       |
| Memory        | ✅                       |
| Request Rate  | ✅                       |
| Response Time | Good for custom metrics |
| Queue Length  | No (use KEDA instead)   |

Recommended production configuration:

```
CPU

Target

70%
```

```
Memory

Target

75%
```

---

# 5.9 Choosing KEDA Metrics

KEDA supports many event sources.

For this architecture:

Use

RabbitMQ Queue Length.

Example

```
Queue < 50

↓

2 Workers

---------------

Queue = 5000

↓

50 Workers

---------------

Queue = 50000

↓

200 Workers
```

Queue length is a better indicator of background workload than CPU utilization.

---

# 5.10 Scaling Boundaries

Every autoscaler should have limits.

Example:

| Component | Min | Max |
| --------- | --- | --- |
| Flask     | 5   | 60  |
| Workers   | 2   | 125 |
| Nodes     | 3   | 12  |

Never leave maximum replicas unlimited.

---

# 5.11 Preventing Oscillation (Flapping)

A common issue is rapid scaling up and down.

Example:

```
CPU

69%

↓

71%

↓

69%

↓

72%
```

Without stabilization:

Pods are constantly created and destroyed.

Use:

* Stabilization windows
* Cooldown periods
* Reasonable polling intervals

Typical values:

| Setting           | Suggested Value |
| ----------------- | --------------- |
| HPA stabilization | 300 sec         |
| KEDA cooldown     | 300 sec         |
| KEDA polling      | 30 sec          |

These should be tuned using real workload patterns.

---

# 5.12 Scaling vs Cost

More pods improve performance but increase cost.

Example:

```
Traffic

↓

10 Pods

↓

$X
```

Peak traffic

↓

```
60 Pods

↓

Higher Cost
```

Autoscaling reduces cost because infrastructure grows only when required.

---

# 5.13 Load Testing

Never rely solely on calculations.

Validate assumptions with load tests.

Recommended scenarios:

### Scenario 1

Normal traffic

```
500 RPS
```

Verify:

* Stable latency
* No unnecessary scaling

---

### Scenario 2

Traffic spike

```
500

↓

5000 RPS
```

Verify:

* HPA scales correctly
* Cluster Autoscaler responds
* Database remains healthy

---

### Scenario 3

Large queue

Publish

```
100,000 Messages
```

Verify:

* KEDA scales workers
* Queue drains
* Workers scale down afterward

---

### Scenario 4

Node failure

Delete an AKS node.

Verify:

* Pods reschedule
* RabbitMQ quorum remains healthy
* No user-visible outage

---

### Scenario 5

Database connection exhaustion

Intentionally lower `max_connections` in a test environment.

Verify:

* Connection pool behavior
* Application error handling
* Alerting

---

# 5.14 Scaling Decision Tree

```text
Incoming HTTP Requests?
        │
        ▼
CPU High?
        │
        ▼
Use HPA
        │
        ▼
Pods Pending?
        │
        ▼
Cluster Autoscaler
```

For asynchronous processing:

```text
Queue Growing?
      │
      ▼
Use KEDA
      │
      ▼
Need More Nodes?
      │
      ▼
Cluster Autoscaler
```

---

# 5.15 Production Recommendations

### Flask API

* Scale with HPA.
* Target CPU and memory.
* Use readiness probes to avoid routing traffic to warming pods.

### Celery Workers

* Scale with KEDA based on queue length.
* Keep tasks idempotent.
* Set sensible `maxReplicaCount` based on database capacity.

### RabbitMQ

* Size queues to absorb bursts.
* Monitor queue depth and consumer lag.
* Use quorum queues for resilience.

### Cluster Autoscaler

* Maintain a small buffer of available capacity if startup latency is critical.
* Separate API and worker node pools to reduce contention.

---

# 5.16 Monitoring Autoscaling

Key dashboards should include:

* Current API pod count
* Current worker pod count
* HPA desired vs current replicas
* KEDA desired vs current replicas
* Pending pods
* Cluster node count
* Queue depth
* API latency
* CPU utilization
* Memory utilization
* Database connections
* Database replica lag

Alerts should trigger when:

* HPA reaches max replicas.
* KEDA reaches max replicas.
* Pending pods persist.
* Queue depth exceeds operational thresholds.
* Database connections exceed 80–90% of capacity.

---

# 5.17 Common Autoscaling Mistakes

| Mistake                         | Consequence               | Better Practice                             |
| ------------------------------- | ------------------------- | ------------------------------------------- |
| Unlimited HPA replicas          | Database overload         | Set max replicas based on connection budget |
| CPU-only worker scaling         | Poor queue responsiveness | Use KEDA with queue metrics                 |
| No resource requests            | Unpredictable scheduling  | Define requests and limits                  |
| One node pool for all workloads | Resource contention       | Use dedicated node pools                    |
| No stabilization window         | Constant scaling          | Configure stabilization/cooldown            |
| Ignoring database limits        | Connection exhaustion     | Capacity-plan from the database outward     |

---

# 5.18 Chapter Summary

Autoscaling is most effective when viewed as a coordinated system rather than independent mechanisms.

* **HPA** reacts to application resource utilization.
* **KEDA** reacts to event-driven workloads.
* **Cluster Autoscaler** provides the infrastructure required by both.

The architecture should always respect downstream limits, especially database capacity. By defining maximum replica counts from the available connection budget and validating those assumptions through load testing, the platform can scale efficiently without sacrificing stability.

A useful mental model is:

```text
Business Demand
        │
        ▼
API Traffic + Queue Growth
        │
        ▼
HPA + KEDA
        │
        ▼
Cluster Autoscaler
        │
        ▼
Database Capacity (Ultimate Constraint)
```

---

# Architect's Notes (Real-World Experience)

One lesson repeated across many production systems is that **the bottleneck changes over time**.

* At low traffic, the API may be CPU-bound.
* As traffic grows, the database often becomes the limiting factor.
* Later, queue throughput, storage IOPS, or external APIs may become the bottleneck.

Because of this, scaling should never be treated as "set and forget." It is an ongoing engineering activity involving measurement, tuning, and capacity planning. Teams that continuously observe metrics, revisit sizing assumptions, and load-test before major releases are far less likely to encounter unexpected production failures.

---

## Next Chapter

The next chapter, **Part 6 – Database Architecture & Optimization**, will focus entirely on designing a highly available, high-performance Azure MySQL deployment. We'll cover schema design, indexing strategies, connection pooling, transaction isolation, read/write splitting, replication lag, backup and recovery, query optimization, and operational practices for running relational databases at scale.
