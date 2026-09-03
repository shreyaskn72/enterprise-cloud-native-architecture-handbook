# Part 9 – Failure Scenarios

## Overview

A production architecture must be designed not only for normal operation, but also for component failures.

This chapter describes how the reference architecture detects failures, recovers from them, and minimizes application impact.

The general recovery model is:

```text
Failure
   ↓
Detection
   ↓
Recovery Action
   ↓
Health Validation
   ↓
Traffic / Workload Recovery
   ↓
Monitoring
```

---

# 9.1 Failure Scenario Summary

| Failure | Detection | Recovery |
|---------|-----------|----------|
| API Pod Crash | Health probes / Kubernetes | Container or Pod restart |
| RabbitMQ Node Failure | Cluster / quorum detection | Quorum recovery |
| Primary DB Failure | Azure HA monitoring | HA failover |
| AKS Node Failure | Node health monitoring | Pod rescheduling |
| Worker Pod Failure | Kubernetes / KEDA | Pod restart |
| Poison Message | Task failure / retry count | DLQ |
| Database Connection Failure | Application health / timeout | Retry / reconnect |
| Queue Backlog | Queue metrics | KEDA scale-out |

---

# 9.2 API Pod Crash

## Scenario

A Flask API pod crashes because of:

- Application exception
- Out-of-memory condition
- Container failure
- Unexpected process termination

```text
Flask API Pod
     │
     X
   Crash
```

## Detection

Kubernetes detects the unhealthy container through:

- Process termination
- Liveness probe failure
- Readiness probe failure

## Recovery Flow

```text
API Pod Failure
      │
      ▼
Kubernetes Detection
      │
      ▼
Container Restart
      │
      ▼
Startup Probe
      │
      ▼
Readiness Probe
      │
      ▼
Pod Ready
      │
      ▼
Service Receives Traffic
```

If the pod cannot recover, Kubernetes may recreate the pod.

## Impact

Requests currently being processed by the failed pod may fail.

New requests should be routed to healthy pods.

```text
          Service
             │
      ┌──────┼──────┐
      ▼      ▼      ▼
   Pod 1   Pod 2   Pod 3
             X
          Failure
```

## Mitigation

- Multiple API replicas
- Readiness probe
- Liveness probe
- Startup probe
- Resource requests and limits
- PodDisruptionBudget
- HPA
- Anti-affinity / topology spreading

---

# 9.3 RabbitMQ Node Failure

## Scenario

One RabbitMQ node becomes unavailable.

```text
RabbitMQ Cluster

Node 1   Node 2   Node 3
  │        X        │
           Failure
```

## Detection

RabbitMQ cluster membership and quorum mechanisms detect the failed node.

For quorum queues, the remaining members can maintain queue availability when quorum is preserved.

## Recovery Flow

```text
RabbitMQ Node Failure
        │
        ▼
Failure Detection
        │
        ▼
Quorum Evaluation
        │
        ▼
Leader Election
        │
        ▼
Remaining Nodes Continue
        │
        ▼
Messages Continue Processing
```

## Impact

The impact depends on:

- Queue type
- Queue quorum
- Cluster health
- Client connection behavior

Clients may need to reconnect after a node failure.

## Mitigation

- Three-node RabbitMQ cluster
- Quorum queues for critical workloads
- Durable queues
- Persistent messages
- Client connection recovery
- RabbitMQ monitoring
- Alerting on node and quorum health

---

# 9.4 Primary Database Failure

## Scenario

The MySQL primary becomes unavailable.

```text
MySQL Primary
      X
   Failure
```

## Detection

The managed database platform detects the failure.

## Recovery

For Azure Database for MySQL Flexible Server configured with HA:

```text
Primary Failure
      │
      ▼
HA Detection
      │
      ▼
Standby Becomes Primary
      │
      ▼
Database Service Recovers
      │
      ▼
Application Reconnects
```

The HA standby is separate from read replicas.

```text
             MySQL
               │
       ┌───────┴────────┐
       │                │
   HA Standby       Read Replicas
   (HA Failover)    (Read Scaling)
```

Read replicas are asynchronous and are not automatically promoted as part of the normal HA standby failover mechanism.

## Application Considerations

Applications should handle:

- Existing connection failures
- Connection retries
- Connection pool refresh
- Temporary request failures

## Mitigation

- Azure Flexible Server HA
- Connection retry logic
- Connection timeout configuration
- Health monitoring
- Backup and recovery strategy
- Tested failover procedures

---

# 9.5 Read Replica Failure

## Scenario

One read replica becomes unavailable.

```text
              Primary
                 │
        ┌────────┼────────┐
        ▼        ▼        ▼
    Replica 1 Replica 2 Replica 3
                  X
               Failure
```

## Recovery

The read routing layer should stop sending traffic to the unhealthy replica.

```text
Replica Failure
      │
      ▼
Health Detection
      │
      ▼
Remove Replica from Read Traffic
      │
      ▼
Route Reads to Healthy Replicas
```

The primary continues handling writes.

## Mitigation

- Health-aware read routing
- Multiple read replicas
- Replication-lag monitoring
- Capacity headroom on remaining replicas

---

# 9.6 AKS Node Failure

## Scenario

An AKS worker node becomes unavailable.

```text
AKS Node

┌─────────────────┐
│ API Pod         │
│ Worker Pod      │
│ Other Pods      │
└─────────────────┘
        X
     Failure
```

## Detection

Kubernetes detects that the node is unhealthy.

## Recovery Flow

```text
Node Failure
      │
      ▼
Node Health Detection
      │
      ▼
Node Marked Unhealthy
      │
      ▼
Pods Become Evictable / Unavailable
      │
      ▼
Scheduler Finds Capacity
      │
      ▼
Pods Rescheduled
      │
      ▼
Readiness Checks
      │
      ▼
Traffic / Processing Resumes
```

If the cluster does not have sufficient capacity, Cluster Autoscaler may add nodes depending on configuration and pending pod requirements.

## Mitigation

- Multiple nodes
- Multiple availability zones where supported
- PodDisruptionBudgets
- Topology spread constraints
- Pod anti-affinity
- Cluster Autoscaler
- Separate node pools

---

# 9.7 Celery Worker Failure

## Scenario

A Celery worker pod crashes while processing tasks.

```text
Celery Worker
     │
     X
   Crash
```

## Recovery

```text
Worker Failure
      │
      ▼
Kubernetes Detection
      │
      ▼
Worker Pod Restart
      │
      ▼
Queue Remains Available
      │
      ▼
Healthy Worker Consumes Tasks
```

Depending on acknowledgement configuration, an unacknowledged task may become available for redelivery.

## Mitigation

- Multiple worker replicas
- Appropriate acknowledgement strategy
- Idempotent tasks
- Retry handling
- KEDA autoscaling
- Worker health monitoring

---

# 9.8 Poison Message

## Scenario

A message repeatedly fails because of a permanent problem.

Example:

```text
Invalid Payload
Invalid Data
Unsupported Operation
```

Without protection:

```text
Queue
 ↓
Worker
 ↓
Failure
 ↓
Retry
 ↓
Worker
 ↓
Failure
 ↓
Retry
```

This can consume worker capacity indefinitely.

## Recovery

```text
Task Failure
      │
      ▼
Retry
      │
      ▼
Exponential Backoff
      │
      ▼
Maximum Retry Count
      │
      ▼
DLQ
```

## Mitigation

- Maximum retry count
- Exponential backoff
- Jitter
- DLQ
- DLQ monitoring
- Controlled replay

---

# 9.9 Database Connection Failure

## Scenario

An API or worker cannot establish a connection to MySQL.

Possible causes:

- Database failover
- Network interruption
- Connection exhaustion
- Temporary database unavailability

## Recovery Flow

```text
Connection Failure
       │
       ▼
Application Detects Error
       │
       ▼
Retry / Reconnect
       │
       ▼
Connection Pool Refresh
       │
       ▼
Database Available
       │
       ▼
Request / Task Continues
```

Retries should use bounded retry counts and backoff.

## Mitigation

- Connection pooling
- Connection timeouts
- Retry with backoff
- Connection pool limits
- Database monitoring
- Sufficient `max_connections` headroom

---

# 9.10 Queue Backlog

## Scenario

The incoming message rate becomes higher than worker processing capacity.

Example:

```text
Incoming = 5,000 msg/sec

Processing = 3,000 msg/sec
```

Queue growth:

```text
5000 - 3000

=

2,000 msg/sec
```

## Recovery Flow

```text
Queue Depth Increases
        │
        ▼
KEDA Detects Queue Length
        │
        ▼
Worker Scale-Out
        │
        ▼
Processing Capacity Increases
        │
        ▼
Queue Drains
        │
        ▼
Workers Scale Down
```

## Mitigation

- KEDA
- Maximum worker limits
- Queue depth alerts
- Worker throughput monitoring
- RabbitMQ capacity planning

---

# 9.11 Cascading Failure Prevention

Failures can propagate between components.

Example:

```text
Database Slow
     │
     ▼
API Requests Become Slow
     │
     ▼
More API Connections
     │
     ▼
Connection Pool Exhaustion
     │
     ▼
API Errors
```

A similar problem can occur with workers:

```text
Database Slow
     │
     ▼
Workers Process Slowly
     │
     ▼
RabbitMQ Queue Grows
     │
     ▼
KEDA Scales Workers
     │
     ▼
More DB Connections
     │
     ▼
Database Overload
```

Therefore, autoscaling must be combined with **database connection limits and downstream capacity**.

This is an important principle:

> **Scaling one layer without considering downstream capacity can amplify a failure instead of fixing it.**

---

# 9.12 Failure Mitigation Strategy

| Layer | Failure Protection |
|-------|--------------------|
| Flask API | Multiple replicas + probes + HPA |
| Celery | Multiple workers + KEDA |
| RabbitMQ | 3-node cluster + quorum queues |
| MySQL | HA + read replicas + backups |
| AKS | Multiple nodes + Cluster Autoscaler |
| Network | Health checks + timeouts + retries |
| Tasks | Idempotency + retry + DLQ |
| Database | Pool limits + retry + monitoring |

---

# 9.13 Failure Detection and Recovery Matrix

| Component | Detect | Recover | Verify |
|-----------|--------|---------|--------|
| API Pod | Health Probe | Restart / Reschedule | Readiness |
| Worker Pod | Kubernetes | Restart / Reschedule | Worker Health |
| RabbitMQ Node | Cluster | Quorum Recovery | Cluster Health |
| MySQL Primary | Azure HA | HA Failover | DB Connectivity |
| Read Replica | Health / Lag | Remove from Routing | Replica Health |
| AKS Node | Node Health | Reschedule Pods | Pod Readiness |
| Queue | Queue Metrics | KEDA Scale-Out | Queue Drain |
| Task | Task Failure | Retry / DLQ | Task Status |

---

# 9.14 Production Failure Testing

Failure recovery should not exist only on paper.

Recommended tests include:

```text
Terminate API Pod
       ↓
Verify automatic recovery
```

```text
Terminate Worker Pod
       ↓
Verify task recovery
```

```text
Simulate RabbitMQ Node Failure
       ↓
Verify quorum recovery
```

```text
Simulate Database Failover
       ↓
Verify application reconnection
```

```text
Drain / Fail an AKS Node
       ↓
Verify pod rescheduling
```

The objective is to validate:

- Detection time
- Recovery time
- Data integrity
- Message integrity
- User impact
- Alerting
- Operational procedures

---

# 9.15 Failure Recovery Principles

| Principle | Description |
|-----------|-------------|
| Detect Quickly | Health checks and monitoring should identify failures early. |
| Fail Safely | Avoid cascading failures and uncontrolled retries. |
| Recover Automatically | Kubernetes and managed Azure services should handle common failures automatically. |
| Preserve Data | Durable messaging and database recovery mechanisms protect state. |
| Retry Carefully | Use bounded retries with backoff. |
| Isolate Failures | DLQs, replicas, and node pools prevent failures from spreading. |
| Test Recovery | Regularly validate failure scenarios. |
| Monitor Recovery | Recovery itself should be observable. |

---

# Chapter Summary

The architecture is designed around the principle that **individual component failures should not become complete application failures**.

The overall resilience model is:

```text
             Failure
                │
                ▼
            Detection
                │
                ▼
        Automatic Recovery
                │
        ┌───────┴────────┐
        ▼                ▼
    Kubernetes         Azure
        │                │
        ▼                ▼
  Restart/Reschedule  Failover
        │                │
        └───────┬────────┘
                ▼
          Health Validation
                │
                ▼
        Service Recovery
                │
                ▼
           Monitoring
```

The key architectural principle is:

> **Design for failure, automate recovery where possible, and continuously test that recovery actually works.**

---

## Next Chapter

➡️ **Part 10 – Monitoring & Observability**