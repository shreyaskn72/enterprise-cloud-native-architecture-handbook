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
Required Capacity

=

Peak Capacity × Growth Factor × Failure Factor
```

Example

Current Peak

```text
5,000 Requests/sec
```

Expected Growth

```text
2x
```

Failure Buffer

```text
20%
```

Required Capacity

```text
5000 × 2 × 1.2

=

12,000 Requests/sec
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
Required API Pods

=

Peak Requests/sec

/

Requests handled per Pod
```

### Example

Peak traffic:

```text
6,000 Requests/sec
```

Each Flask pod can safely process:

```text
120 Requests/sec
```

Calculation:

```text
6000 / 120

=

50 Pods
```

Add a 20% safety margin:

```text
50 × 1.2

=

60 Pods
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
Incoming Traffic

↓

CPU > 70%

↓

HPA

↓

More Flask Pods

↓

Cluster Autoscaler (if needed)

↓

Additional AKS Nodes
```

---

# 6.6 Background Worker Capacity

## Formula

```text
Workers Required

=

Incoming Jobs/sec

/

Jobs processed/sec by one Worker
```

Example

Incoming jobs:

```text
2,500 Jobs/sec
```

Each worker processes:

```text
25 Jobs/sec
```

Calculation:

```text
2500 / 25

=

100 Workers
```

Add safety margin:

```text
100 × 1.25

=

125 Workers
```

Recommended KEDA limits:

| Setting     | Value |
| ----------- | ----- |
| Min Workers | 2     |
| Max Workers | 125   |

---

# 6.7 RabbitMQ Queue Planning

Queues should absorb temporary spikes.

Example:

Traffic spike:

```text
10,000 Jobs

↓

RabbitMQ Queue

↓

Workers process gradually
```

Queue depth formula:

```text
Maximum Queue

=

Maximum Processing Delay

×

Incoming Jobs/sec
```

Example:

Maximum acceptable delay:

```text
30 seconds
```

Incoming jobs:

```text
2,000/sec
```

Queue size:

```text
30 × 2000

=

60,000 messages
```

This becomes an operational threshold for alerts.

---

# 6.8 Database Capacity Planning

The database is usually the first bottleneck in a scalable architecture.

Everything else must be designed around it.

---

## Step 1 — Maximum Database Connections

Assume Azure MySQL supports:

```text
Max Connections

=

1,200
```

Reserve:

| Purpose              | Connections |
| -------------------- | ----------- |
| DBA/Admin            | 40          |
| Monitoring           | 30          |
| Backup & Maintenance | 30          |

Reserved:

```text
100
```

Application budget:

```text
1200 - 100

=

1100 Connections
```

---

## Step 2 — API Pool

Suppose:

Maximum API Pods

```text
60
```

Pool size:

```text
10
```

Formula

```text
Connections

=

Pods × Pool Size
```

Result

```text
60 × 10

=

600 Connections
```

---

## Step 3 — Celery Pool

Maximum Workers

```text
100
```

Pool

```text
5
```

Connections

```text
100 × 5

=

500 Connections
```

---

## Step 4 — Validation

Total application connections

```text
600

+

500

=

1100
```

Exactly matches our planned budget.

---

## Capacity Formula

```text
(API Pods × API Pool)

+

(Workers × Worker Pool)

+

Reserved

≤

Database Maximum Connections
```

This should always be validated before changing HPA or KEDA limits.

---

# 6.9 Read Replica Planning

Example workload:

```text
Reads

=

80%

Writes

=

20%
```

Architecture:

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

# 6.10 AKS Node Planning

Assume:

Each node supports:

```text
30 Pods
```

Total workload:

* API Pods = 60
* Worker Pods = 125
* RabbitMQ = 3
* Celery Beat = 1
* Ingress = 2
* Monitoring = 10
* System Pods = 15

Total:

```text
216 Pods
```

Formula

```text
Nodes

=

Pods

/

Pods per Node
```

Calculation

```text
216 / 30

=

7.2
```

Round up:

```text
8 Nodes
```

---

# 6.11 Autoscaling Limits

Recommended limits:

| Component      | Min | Max |
| -------------- | --- | --- |
| Flask API      | 5   | 60  |
| Celery Workers | 2   | 125 |
| RabbitMQ       | 3   | 3   |
| Celery Beat    | 1   | 1   |
| Cluster Nodes  | 3   | 10  |

These values should be revisited after load testing.

---

# 6.12 Resource Requests and Limits

Example Kubernetes sizing:

| Component     | CPU Request | CPU Limit | Memory Request | Memory Limit |
| ------------- | ----------- | --------- | -------------- | ------------ |
| Flask API     | 500m        | 1 CPU     | 512 MiB        | 1 GiB        |
| Celery Worker | 500m        | 2 CPU     | 1 GiB          | 2 GiB        |
| RabbitMQ      | 2 CPU       | 4 CPU     | 4 GiB          | 8 GiB        |
| Celery Beat   | 250m        | 500m      | 256 MiB        | 512 MiB      |

These values should be tuned based on profiling rather than guessed.

---

# 6.13 Capacity Validation Checklist

Before production, verify:

* ✔ Peak RPS is known.
* ✔ Peak queue throughput is known.
* ✔ Average and P95 API latency are measured.
* ✔ Average Celery task duration is measured.
* ✔ Database `max_connections` is documented.
* ✔ Connection pools are sized to stay within the database budget.
* ✔ HPA maximum replicas respect database capacity.
* ✔ KEDA maximum replicas respect database capacity.
* ✔ AKS node limits can accommodate the maximum pod count.
* ✔ Load tests validate all assumptions.

---

# 6.14 Chapter Summary

Capacity planning is about ensuring every layer scales together without overwhelming another.

The key dependency chain is:

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

The database often defines the practical upper limit for the entire system. By calculating connection budgets first and then deriving HPA and KEDA limits from that budget, the platform can scale predictably without exhausting shared resources.

---

# Next Chapter

➡ **Part 7 – Database Design**
