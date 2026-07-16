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

Each autoscaler has a different responsibility and works independently but cooperatively:

| Component | Trigger | Purpose | Scales |
|-----------|---------|---------|--------|
| **HPA** | CPU / Memory / Custom Metrics | Replicates API pods | Pod replicas |
| **KEDA** | Queue length / Events | Replicates workers | Worker replicas |
| **Cluster Autoscaler** | Pending pods | Adds compute capacity | AKS nodes |

**Request Path Scaling:**
```
Traffic ↑ → HPA detects CPU ↑ → Creates more pods → Nodes full? → Cluster Autoscaler adds nodes
```

**Background Job Scaling:**
```
Queue ↑ → KEDA detects queue length ↑ → Creates more workers → Nodes full? → Cluster Autoscaler adds nodes
```

---

# 5.3 Request Scaling Flow (Step by Step)

## Step 1: Traffic Increases
Incoming request rate jumps from 1,000 RPS → 6,000 RPS

## Step 2: API CPU Increases
Pod CPU utilization rises from 35% → 85%

## Step 3: HPA Calculates Desired Replicas
**Formula:**
```
Desired Replicas = Current Replicas × (Current Metric / Target Metric)
```

**Example:**
```
Current Pods   = 10
Current CPU    = 84%
Target CPU     = 70%
Desired Pods   = 10 × (84/70) = 12 pods
```
→ HPA creates 2 new pods

## Step 4: Check Node Capacity
If no nodes have available resources, new pods remain in **Pending** state

## Step 5: Cluster Autoscaler Detects Pending Pods
Cluster Autoscaler automatically provisions new nodes → Pods are scheduled

---

# 5.4 Worker Scaling Flow

## Scenario: Queue Message Spike

**Initial State:**
- Queue Length: 100 messages
- Active Workers: 2

**Event:**
- Queue suddenly grows to 25,000 messages

## KEDA Scaling Calculation

**Formula:**
```
Workers Needed = Queue Length / Messages handled per Worker
```

**Example:**
```
Queue Length           = 25,000 messages
Capacity per Worker    = 100 messages
Workers Needed         = 25,000 / 100 = 250 workers
```

**Result:** KEDA requests 250 worker pods. If insufficient node capacity, Cluster Autoscaler adds nodes.

---

# 5.5 Cluster Autoscaler Flow

## Scenario: Node Capacity Exhaustion

**Initial Setup:**
- Worker Pool: 3 nodes
- Max pods per node: 30 pods
- Max capacity: 90 pods

**Event:** KEDA creates 200 worker pods
- 90 can be scheduled on existing nodes
- 110 remain **Pending**

**Cluster Autoscaler Response:**
```
Pending Pods (110) → Assess VM requirements → Add new nodes → Scheduler places pods
```

The autoscaler automatically provisions additional capacity and schedules workloads.

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

Autoscaling is meaningless if the database cannot handle the connection load.

**Example Problem:**

| Component | Max Pods | Pool Size | Connections |
|-----------|----------|-----------|-------------|
| Flask API | 60 | 10 | 600 |
| Workers | 125 | 5 | 625 |
| **Total** | - | - | **1,225** |

If MySQL supports only 1,200 max connections → **Architecture fails**

**Critical Formula:**
```
(API Pods × Pool Size) + (Worker Pods × Pool Size) + Reserved ≤ Database Max Connections
```

**Action:**
✅ Always verify connection budgets before scaling limits  
✅ Leave headroom for admin connections and monitoring  
✅ Monitor actual connection utilization continuously

---

# 5.8 Choosing HPA Metrics

CPU alone is insufficient. Use a combination of metrics for better scaling decisions:

| Metric | Recommendation | Notes |
|--------|----------------|-------|
| **CPU** | ✅ Primary | Good baseline, but can be misleading with bursty workloads |
| **Memory** | ✅ Primary | Indicates application health and potential memory leaks |
| **Request Rate** | ✅ Secondary | Excellent for request-heavy workloads (API servers) |
| **Response Time** | ⚠️ Use cautiously | Good for custom metrics, requires careful threshold tuning |
| **Queue Length** | ❌ No (use KEDA) | Queue scaling is handled by KEDA, not HPA |

**Recommended Production Configuration:**

```yaml
metrics:
- type: Resource
  resource:
    name: cpu
    target:
      averageUtilization: 70
- type: Resource
  resource:
    name: memory
    target:
      averageUtilization: 75
```

**Key Insight:**
Use **70% CPU** and **75% Memory** as targets to leave headroom for spikes while avoiding constant scaling.

---

# 5.9 Choosing KEDA Metrics

KEDA supports many event sources. For this architecture, use **RabbitMQ Queue Length**.

**Why Queue Length?**
- Direct indicator of background workload
- More accurate than CPU for async processing
- Automatic scale-down when queue drains

**Queue Length Scaling Examples:**

| Queue Length | Workers | Rationale |
|--------------|---------|-----------|
| < 50 | 2 | Low load, minimal workers |
| 500-2,000 | 10-20 | Normal operation |
| 5,000 | 50 | Moderate spike |
| 50,000 | 200 | Heavy spike |

**Key Principle:**
Queue length is a **direct reflection of work** that needs to be processed. Scale workers proportionally to drain the queue efficiently.

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

**Problem:** Rapid scaling up and down when metrics hover near the threshold.

**Example:** CPU fluctuates around target:
```
69% → 71% → 69% → 72% → 70% → ...
```

Without stabilization, pods are constantly created/destroyed, wasting resources.

**Solution: Stabilization Windows & Cooldown Periods**

| Setting | Recommended Value | Purpose |
|---------|-------------------|---------|
| **HPA Stabilization** | 300 sec (5 min) | Don't scale down for 5 minutes after scale-up |
| **KEDA Cooldown** | 300 sec (5 min) | Prevent rapid scale-down |
| **KEDA Polling** | 30 sec | Check queue length every 30 sec |

**Benefits:**
✅ Smoother scaling behavior  
✅ Reduced pod churn and node allocations  
✅ Stable resource costs  

**Tuning:** Start with defaults, adjust based on your actual workload patterns.

---

# 5.12 Scaling vs Cost Trade-off

**Principle:** More pods improve performance but increase infrastructure costs.

**Example Cost Comparison:**

| Scenario | Pod Count | Approximate Cost | Behavior |
|----------|-----------|------------------|----------|
| **Off-peak** | 5 | Base cost | Sufficient for low traffic |
| **Normal** | 10-15 | 2x base | Standard operation |
| **Peak** | 60+ | 6-12x base | Traffic spike handling |

**Key Insight:**
Autoscaling **reduces** total cost because infrastructure grows **only when required**. Pay only for what you use.

**Cost Optimization Tips:**
- ✅ Set appropriate max replicas (avoid unlimited scaling)
- ✅ Use cooldown periods to prevent wasteful rapid scaling
- ✅ Review scaling patterns regularly for cost optimization opportunities

---

# 5.13 Validation: Load Testing Scenarios

Never rely solely on calculations. Validate with load tests.

## Scenario 1: Normal Traffic (Baseline)
**Test:** 500 RPS sustained  
**Verify:**
- ✅ Latency is stable and acceptable
- ✅ No unnecessary scaling activity
- ✅ Pods remain at minimum replicas

## Scenario 2: Traffic Spike
**Test:** 500 RPS → 5,000 RPS (10x increase)  
**Verify:**
- ✅ HPA detects and adds pods within 1-2 minutes
- ✅ Cluster Autoscaler provisions nodes if needed
- ✅ Database remains healthy (connections, latency)
- ✅ Application recovers within acceptable time

## Scenario 3: Large Background Queue
**Test:** Publish 100,000 messages to queue  
**Verify:**
- ✅ KEDA scales workers appropriately
- ✅ Queue drains within target time
- ✅ Workers scale down after queue is empty

## Scenario 4: Node Failure
**Test:** Terminate an AKS node during load  
**Verify:**
- ✅ Pods reschedule to healthy nodes
- ✅ RabbitMQ quorum remains intact
- ✅ No user-visible outage

## Scenario 5: Connection Exhaustion
**Test:** Lower `max_connections` in test environment  
**Verify:**
- ✅ Connection pool behavior is correct
- ✅ Application handles gracefully (circuit breaker)
- ✅ Alerts fire appropriately

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

| Component | Scaling Method | Key Practices |
|-----------|-----------------|------|
| **Flask API** | HPA (CPU + Memory) | Use readiness probes; avoid routing traffic to warming pods |
| **Celery Workers** | KEDA (Queue Length) | Keep tasks idempotent; set max replicas based on DB capacity |
| **RabbitMQ** | N/A (Fixed/Clustered) | Size queues for bursts; monitor queue depth |
| **Cluster Autoscaler** | Pending Pods | Maintain small buffer if latency-critical; use separate node pools |

**Critical Rule:** Always set max replicas based on **downstream capacity**, not arbitrary numbers.

---

# 5.16 Monitoring Autoscaling

**Essential Dashboards:**

| Metric | Purpose |
|--------|---------|
| **Pod Counts** | API pod count, worker pod count, current vs desired |
| **HPA Status** | Desired vs actual replicas, scaling events |
| **KEDA Status** | Desired vs actual workers, queue depth |
| **Infrastructure** | Pending pods, node count, available capacity |
| **Performance** | API latency, CPU/Memory utilization |
| **Database Health** | Connection count (80-90% warning), replica lag |

**Alert Thresholds:**

| Alert | Threshold | Action |
|-------|-----------|--------|
| HPA at max | Reached `maxReplicaCount` | Investigate capacity limits |
| KEDA at max | Reached `maxReplicaCount` | Check queue backlog |
| Pending pods | > 0 for 5+ minutes | Cluster Autoscaler slow or stuck |
| Queue depth | Exceeds operational target | Check worker health |
| DB connections | > 90% of max | Risk of connection exhaustion |

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

Autoscaling is most effective when viewed as a **coordinated system**, not independent mechanisms.

**How They Work Together:**
- **HPA** reacts to application resource utilization (CPU/Memory)
- **KEDA** reacts to event-driven workloads (queue length)
- **Cluster Autoscaler** provisions infrastructure for pending pods

**Golden Rule:**
> Scale only as much as downstream systems can safely handle.

The architecture must **respect downstream limits**, especially database connection capacity. Define maximum replica counts from the **available connection budget**, then validate through load testing.

**Mental Model:**
```
Business Demand
        │
        ▼
API Traffic + Queue Growth
        │
        ▼
HPA + KEDA (Pod Scaling)
        │
        ▼
Cluster Autoscaler (Node Provisioning)
        │
        ▼
Database Capacity (Ultimate Constraint)
```

**Key Insight:** The bottleneck changes over time. What's CPU-bound at low traffic becomes database-bound at scale. Autoscaling is not "set and forget"—it requires continuous measurement, tuning, and capacity planning.

---

## Next Chapter

The next chapter, **Part 6 – Capacity Planning**.
