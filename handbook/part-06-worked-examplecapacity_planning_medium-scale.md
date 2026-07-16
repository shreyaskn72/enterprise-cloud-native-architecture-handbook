# Appendix A – Capacity Planning Worked Examples

## Overview

This appendix demonstrates how the capacity planning formulas introduced in **Part 6 – Capacity Planning** can be applied to real-world workloads.

The examples use simplified assumptions for illustration. Actual production sizing should always be validated through performance testing, monitoring, and workload analysis.

---

# Example 1 – API Capacity Planning

## Scenario

A Flask API serves a peak workload of **5,000 requests per second (RPS)**.

Performance testing shows that a single API pod can handle **250 RPS** while maintaining acceptable latency.

## Formula

```text
API Pods Required

=

Peak RPS

/

RPS per Pod
```

## Calculation

```text
5000 / 250

=

20 Pods
```

## Recommended Configuration

| Property | Value |
|----------|------:|
| Minimum Replicas | 5 |
| Maximum Replicas | 30 |
| Target CPU Utilization | 70% |
| Expected Peak Pods | 20 |

## Recommendation

Always provision additional headroom (20–30%) to accommodate sudden traffic spikes and rolling deployments.

---

# Example 2 – Celery Worker Capacity

## Scenario

The application receives **2,000 background jobs per second**.

Performance testing shows that one Celery worker processes **100 jobs per second**.

## Formula

```text
Workers Required

=

Incoming Jobs/sec

/

Worker Throughput
```

## Calculation

```text
2000 / 100

=

20 Workers
```

## Recommended Configuration

| Property | Value |
|----------|------:|
| Minimum Workers | 2 |
| Maximum Workers | 50 |
| Scaling Method | KEDA |
| Trigger | RabbitMQ Queue Length |

## Recommendation

Worker throughput should exceed message arrival rate to prevent continuous queue growth.

---

# Example 3 – RabbitMQ Capacity Planning

## Scenario

RabbitMQ receives **2,000 messages per second**.

Each consumer processes **100 messages per second**.

## Formula

```text
Consumers Required

=

Messages/sec

/

Consumer Throughput
```

## Calculation

```text
2000 / 100

=

20 Consumers
```

## Recommended Configuration

| Property | Value |
|----------|------:|
| RabbitMQ Cluster | 3 Nodes |
| Queue Type | Quorum Queue |
| Consumers | 20 |

## Recommendation

Monitor queue depth continuously. Persistent queue growth indicates that consumer capacity should be increased.

---

# Example 4 – Database Connection Planning

## Scenario

The deployment contains:

- 20 Flask API Pods
- 20 Celery Worker Pods
- API Pool Size = 20
- Worker Pool Size = 10

Additional database connections are reserved for administration and monitoring.

## Formula

```text
Total Connections

=

(API Pods × API Pool)

+

(Worker Pods × Worker Pool)

+

Admin

+

Monitoring
```

## Calculation

```text
(20 × 20)

+

(20 × 10)

+

20

+

10

=

630 Connections
```

## Connection Breakdown

| Source | Connections |
|---------|------------:|
| Flask API | 400 |
| Celery Workers | 200 |
| Administration | 20 |
| Monitoring | 10 |
| **Total** | **630** |

## Recommendation

Configure the database with sufficient headroom.

Example:

```text
Database Maximum Connections = 800–1000
```

This allows for maintenance activities, failover events, and unexpected workload spikes.

---

# Example 5 – Read Replica Sizing

## Scenario

The application processes:

- 20% Write Operations
- 80% Read Operations

Peak workload:

- 5,000 Requests/sec

## Calculation

```text
Writes

=

5000 × 20%

=

1000 RPS
```

```text
Reads

=

5000 × 80%

=

4000 RPS
```

Assume ten read replicas.

```text
Reads per Replica

=

4000 / 10

=

400 RPS
```

## Recommendation

| Component | Workload |
|-----------|---------:|
| Primary Database | 1,000 Write RPS |
| Read Replica | ~400 Read RPS per Replica |

Monitor replication lag to ensure read consistency.

---

# Example 6 – AKS Node Capacity

## Scenario

The deployment consists of:

| Component | Pods |
|-----------|-----:|
| Flask API | 20 |
| Celery Workers | 20 |
| Celery Beat | 1 |
| RabbitMQ | 3 |
| Monitoring | 6 |
| Ingress | 2 |
| System Components | 8 |

## Total Pods

```text
20 + 20 + 1 + 3 + 6 + 2 + 8

=

60 Pods
```

Assume each node supports **30 Pods**.

## Formula

```text
Nodes Required

=

Pods

/

Pods Per Node
```

## Calculation

```text
60 / 30

=

2 Nodes
```

## Production Recommendation

To provide high availability and support rolling upgrades, dedicate node pools by workload.

| Node Pool | Recommended Nodes |
|------------|------------------:|
| System | 2 |
| API | 2 |
| Worker | 2 |
| Messaging | 3 |
| **Total** | **9 Nodes** |

---

# Example 7 – End-to-End Capacity Summary

| Component | Recommended Capacity |
|-----------|---------------------:|
| Peak API Traffic | 5,000 RPS |
| Flask API Pods | 20 |
| Celery Worker Pods | 20 |
| Celery Beat | 1 |
| RabbitMQ Nodes | 3 |
| MySQL Primary | 1 |
| Read Replicas | 10 |
| Database Connections | 630 |
| AKS Nodes | 9 |

---

# Key Takeaways

- Always size infrastructure using **peak expected workload**.
- Maintain **20–30% operational headroom** for failover, deployments, and traffic spikes.
- Validate assumptions using **load testing** before production deployment.
- Continuously monitor utilization and adjust capacity as application demand evolves.
- Capacity planning is an iterative process and should be reviewed regularly as workloads grow.