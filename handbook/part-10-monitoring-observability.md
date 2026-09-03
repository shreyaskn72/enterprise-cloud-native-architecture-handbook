# Part 10 – Monitoring & Observability

> **Goal:** Build an observability system that helps engineers answer three questions quickly:
>
> 1. **Is the system healthy?**
> 2. **Where is the problem?**
> 3. **What is the impact on users?**

---

## 10.1 Observability Overview

A production system generates three primary types of telemetry:

| Telemetry   | Purpose                           | Example                       |
| ----------- | --------------------------------- | ----------------------------- |
| **Metrics** | What is happening?                | CPU = 80%, RPS = 500          |
| **Logs**    | What happened?                    | `Database connection timeout` |
| **Traces**  | Where did the request spend time? | API → DB → RabbitMQ           |

Our architecture uses:

```text
                         ┌──────────────────────┐
                         │    Applications      │
                         │                      │
                         │ Flask API            │
                         │ Celery Workers       │
                         │ Celery Beat          │
                         │ RabbitMQ              │
                         └──────────┬───────────┘
                                    │
                   ┌────────────────┼────────────────┐
                   │                │                │
                   ▼                ▼                ▼
              Metrics            Logs             Traces
                   │                │                │
                   ▼                ▼                ▼
              Prometheus          Loki        OpenTelemetry
                   │                │                │
                   └────────────┬───┴────────────────┘
                                │
                                ▼
                           ┌──────────┐
                           │ Grafana  │
                           └────┬─────┘
                                │
                                ▼
                     Dashboards + Alerts

                  Azure Resources
                        │
                        ▼
                  Azure Monitor
```

### Key Principle

**Monitoring tells us that something is wrong.
Observability helps us understand why.**

---

# 10.2 Observability Stack

## 10.2.1 Prometheus – Metrics Collection

Prometheus collects and stores time-series metrics.

Typical metrics:

```text
API requests/sec
API request latency
API error rate
Pod CPU
Pod memory
Worker task count
RabbitMQ queue depth
RabbitMQ message rate
Database connections
Database CPU
Database storage
```

Example:

```text
api_requests_total
api_request_duration_seconds
api_errors_total

celery_tasks_total
celery_task_duration_seconds

rabbitmq_queue_messages
rabbitmq_queue_consumers

container_cpu_usage
container_memory_usage
```

### Why Prometheus?

Prometheus is well suited for Kubernetes because it can dynamically discover workloads and scrape metrics from applications and infrastructure.

---

# 10.2.2 Grafana – Visualization

Grafana provides dashboards for metrics and other observability data.

Typical dashboards:

```text
┌─────────────────────────────────────────────┐
│              API Dashboard                  │
├─────────────────────────────────────────────┤
│ RPS       │ Error Rate │ P95 Latency       │
│ 1,250     │ 0.3%       │ 180 ms            │
├─────────────────────────────────────────────┤
│                                             │
│ Request Rate        Latency                │
│     /\                 /\                  │
│    /  \               /  \                 │
│___/    \_____________/    \____             │
│                                             │
├─────────────────────────────────────────────┤
│ Pod CPU │ Pod Memory │ Restart Count       │
└─────────────────────────────────────────────┘
```

Recommended dashboards:

* API Dashboard
* Kubernetes Dashboard
* Celery Worker Dashboard
* RabbitMQ Dashboard
* MySQL Dashboard
* Infrastructure Dashboard
* SLO Dashboard

---

# 10.2.3 Loki – Log Aggregation

Loki aggregates application and infrastructure logs.

Example application log:

```text
2026-09-03T10:15:23Z
level=ERROR
service=flask-api
trace_id=abc123
message="Database connection timeout"
```

Instead of logging only:

```text
Database connection timeout
```

include useful context:

```text
timestamp
level
service
pod
namespace
trace_id
request_id
message
```

This makes troubleshooting much easier.

### Structured Logging

Prefer structured logs:

```json
{
  "level": "ERROR",
  "service": "flask-api",
  "trace_id": "abc123",
  "request_id": "req789",
  "message": "Database connection timeout"
}
```

This allows logs to be searched and filtered efficiently.

---

# 10.2.4 Azure Monitor

Azure Monitor provides cloud-native monitoring for Azure resources.

It can monitor resources such as:

```text
AKS
Azure Database for MySQL
Load Balancer
Application Gateway
Storage
Networking
```

Example signals:

```text
AKS node CPU
AKS node memory
MySQL CPU
MySQL connections
MySQL storage
Network metrics
Azure resource health
```

### Prometheus vs Azure Monitor

They serve complementary purposes.

| Prometheus                     | Azure Monitor                       |
| ------------------------------ | ----------------------------------- |
| Kubernetes/application metrics | Azure resource monitoring           |
| Application-centric            | Azure platform-centric              |
| PromQL                         | Azure monitoring/query capabilities |
| Kubernetes ecosystem           | Azure ecosystem                     |
| Custom application metrics     | Azure service metrics               |

A production architecture can use both.

---

# 10.2.5 OpenTelemetry – Distributed Tracing

OpenTelemetry provides a standard way to generate and collect telemetry, especially distributed traces.

Example request:

```text
User
 │
 ▼
Ingress
 │
 ▼
Flask API
 │
 ├──────► MySQL
 │
 └──────► RabbitMQ
             │
             ▼
        Celery Worker
             │
             ▼
           MySQL
```

Without tracing, we may only see:

```text
API latency = 2 seconds
```

With tracing:

```text
Total request = 2 seconds

Ingress       = 20 ms
Flask API     = 100 ms
MySQL         = 1,700 ms
RabbitMQ      = 50 ms
Other         = 130 ms
```

Now the bottleneck is immediately visible.

---

# 10.3 Metrics Collection Architecture

A simplified flow:

```text
                    ┌───────────────┐
                    │ Flask API     │
                    └───────┬───────┘
                            │
                            │ /metrics
                            ▼
                    ┌───────────────┐
                    │ Prometheus    │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │    Grafana    │
                    └───────────────┘


RabbitMQ ───────────────► Prometheus
Kubernetes ─────────────► Prometheus
MySQL ──────────────────► Prometheus
```

Prometheus periodically scrapes metrics from configured targets.

---

# 10.4 What Should We Monitor?

Monitoring should follow the architecture.

## API

Monitor:

```text
Request rate
Error rate
Latency
HTTP status codes
Active requests
Pod CPU
Pod memory
Pod restarts
```

## Celery Workers

Monitor:

```text
Tasks received
Tasks completed
Tasks failed
Task duration
Active tasks
Worker count
Queue backlog
Worker CPU
Worker memory
```

## RabbitMQ

Monitor:

```text
Queue depth
Messages published
Messages consumed
Unacknowledged messages
Consumer count
Publish rate
Consume rate
Node health
Disk usage
Memory usage
```

## MySQL

Monitor:

```text
CPU
Memory
Connections
Connection utilization
Query latency
Slow queries
Replication lag
Storage
IOPS
Transaction rate
```

## Kubernetes

Monitor:

```text
Node CPU
Node memory
Pod CPU
Pod memory
Pod restarts
Pending pods
Failed pods
Deployment availability
HPA status
KEDA status
PVC status
```

---

# 10.5 Custom Dashboards

A good dashboard should answer a specific operational question.

### API Dashboard

```text
┌──────────────────────────────────────────┐
│ API HEALTH                               │
├──────────────────────────────────────────┤
│ RPS             1,250                    │
│ Error Rate      0.30%                    │
│ P50 Latency     80 ms                    │
│ P95 Latency     180 ms                   │
│ P99 Latency     450 ms                   │
├──────────────────────────────────────────┤
│ Requests / sec                           │
│ Latency                                  │
│ Errors                                   │
├──────────────────────────────────────────┤
│ Healthy Pods: 8 / 8                      │
└──────────────────────────────────────────┘
```

### Queue Dashboard

```text
┌──────────────────────────────────────────┐
│ RABBITMQ / CELERY                        │
├──────────────────────────────────────────┤
│ Queue Depth          2,450                │
│ Publish Rate         800/sec              │
│ Consume Rate         750/sec              │
│ Consumers            12                   │
│ Unacked Messages     300                  │
├──────────────────────────────────────────┤
│ Queue Growth                              │
│ Worker Throughput                         │
└──────────────────────────────────────────┘
```

### Database Dashboard

```text
┌──────────────────────────────────────────┐
│ MYSQL                                    │
├──────────────────────────────────────────┤
│ CPU                  62%                  │
│ Connections          720 / 1,000          │
│ Query Latency        45 ms                │
│ Slow Queries         12/min               │
│ Replication Lag      1.2 sec              │
├──────────────────────────────────────────┤
│ Connection Usage                         │
│ Query Latency                            │
│ Replication Lag                          │
└──────────────────────────────────────────┘
```

---

# 10.6 Alerting

Dashboards are useful when engineers are looking at them.

**Alerts are required when engineers are not.**

The alerting pipeline:

```text
Metric
  │
  ▼
Prometheus
  │
  ▼
Alert Rule
  │
  ▼
Alert Manager
  │
  ├────► Email
  ├────► Teams
  ├────► Slack
  └────► Pager / On-call
```

---

# 10.7 Alert Design

Avoid alerting on every small fluctuation.

Bad:

```text
CPU > 70%
```

for a few seconds.

Better:

```text
CPU > 85%
for 10 minutes
```

Even better:

```text
CPU > 85%
AND
request latency is increasing
```

Alerts should represent **actionable conditions**.

---

# 10.8 Recommended Alerts

### API

```text
Error rate > 5%
P95 latency > SLO
No healthy API pods
High pod restart rate
```

### RabbitMQ

```text
Queue depth continuously increasing
High unacknowledged messages
No consumers
RabbitMQ node unavailable
Disk usage critical
```

### Celery

```text
Worker count below minimum
Task failure rate high
Task latency high
Queue backlog growing
```

### MySQL

```text
Connection utilization high
CPU continuously high
Storage almost full
Slow query rate high
Replication lag high
Database unavailable
```

### Kubernetes

```text
Node unavailable
Pods pending
Pods crash looping
Deployment unavailable
PVC pending
HPA unable to scale
```

---

# 10.9 SLO Framework

Monitoring tells us what is happening.

SLOs define **how good the service should be**.

The three important concepts are:

```text
SLI
 │
 │ Measures actual performance
 ▼
SLO
 │
 │ Defines desired performance
 ▼
Error Budget
 │
 │ Defines acceptable failure
 ▼
Engineering Decisions
```

---

# 10.10 SLI – Service Level Indicator

An SLI is a measurable indicator of service quality.

Examples:

```text
Availability
Latency
Error rate
Successful requests
Queue processing time
```

Example:

```text
Successful Requests
──────────────────── × 100
Total Requests
```

If:

```text
Total requests     = 1,000,000
Successful         =   999,000
```

Then:

```text
Availability SLI = 99.9%
```

---

# 10.11 SLO – Service Level Objective

An SLO defines the target.

Example:

```text
API Availability SLO = 99.9%

API P95 Latency SLO = < 300 ms

Background Job Success SLO = 99.5%
```

Example:

```text
SLI:
99.94% successful requests

SLO:
99.90%

Result:
SLO achieved
```

---

# 10.12 Error Budget

Error budget represents how much failure is acceptable.

Formula:

```text
Error Budget = 100% - SLO
```

For:

```text
SLO = 99.9%
```

we get:

```text
Error Budget = 0.1%
```

For one 30-day month:

```text
30 × 24 × 60
= 43,200 minutes
```

Allowed downtime:

```text
43,200 × 0.001
= 43.2 minutes
```

So a 99.9% availability SLO allows approximately:

**43.2 minutes of unavailability per 30-day period.**

---

# 10.13 Error Budget Management

Error budget should influence engineering decisions.

```text
             Error Budget
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
   Budget healthy       Budget exhausted
        │                   │
        ▼                   ▼
   New releases        Prioritize reliability
   Feature work        Fix major incidents
   Experiments         Reduce deployment risk
```

Example:

```text
SLO = 99.9%
Error budget = 43.2 minutes

Used = 10 minutes
Remaining = 33.2 minutes
```

The service still has budget available.

If most of the budget is consumed:

```text
Remaining budget = 2 minutes
```

the team should prioritize reliability work instead of aggressively introducing risky changes.

---

# 10.14 Golden Signals

The four Golden Signals are:

```text
Latency
Traffic
Errors
Saturation
```

## Latency

How long requests take.

Example:

```text
P50 = 80 ms
P95 = 180 ms
P99 = 450 ms
```

Focus especially on percentiles rather than only averages.

---

## Traffic

How much demand the system receives.

Examples:

```text
Requests/sec
Messages/sec
Tasks/sec
Database queries/sec
```

---

## Errors

How many requests or operations are failing.

Examples:

```text
HTTP 5xx
Task failures
Database errors
Message processing failures
Timeouts
```

---

## Saturation

How close a resource is to its capacity.

Examples:

```text
CPU = 85%
Memory = 90%
DB connections = 90%
RabbitMQ disk = 80%
Queue depth increasing
```

### Golden Signals

```text
             Service Health
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
    Latency     Traffic    Errors
       │
       └──────────► Saturation
```

---

# 10.15 RED Methodology

RED is especially useful for request-driven services such as Flask APIs.

```text
R = Rate
E = Errors
D = Duration
```

## Rate

How many requests are being handled?

```text
Requests/sec = 1,250
```

## Errors

How many requests are failing?

```text
Error rate = 0.3%
```

## Duration

How long requests take?

```text
P95 = 180 ms
P99 = 450 ms
```

### API Dashboard Using RED

```text
┌──────────────────────────────────────┐
│              API - RED               │
├──────────────────────────────────────┤
│ Rate       1,250 req/sec             │
│ Errors     0.3%                      │
│ Duration   P95 = 180 ms              │
└──────────────────────────────────────┘
```

---

# 10.16 USE Methodology

USE is particularly useful for infrastructure and resources.

```text
U = Utilization
S = Saturation
E = Errors
```

## Utilization

How much of the resource is being used?

```text
CPU = 75%
Memory = 68%
Disk = 70%
```

## Saturation

How much work is waiting because the resource is constrained?

Examples:

```text
CPU run queue
Disk IO queue
RabbitMQ queue backlog
DB connection wait
```

## Errors

Resource-level failures.

Examples:

```text
Disk errors
Network errors
Node failures
IO failures
```

---

# 10.17 RED vs USE

| Method  | Best For                   | Focus                           |
| ------- | -------------------------- | ------------------------------- |
| **RED** | Applications / APIs        | Rate, Errors, Duration          |
| **USE** | Infrastructure / Resources | Utilization, Saturation, Errors |

Example:

```text
Flask API
    │
    └──► RED

AKS Node
    │
    └──► USE

MySQL
    │
    └──► RED + USE

RabbitMQ
    │
    └──► RED + USE
```

---

# 10.18 End-to-End Troubleshooting Example

Suppose users report:

```text
"The API is slow."
```

Start with the Golden Signals:

```text
Traffic   = normal
Errors    = normal
Latency   = HIGH
Saturation = HIGH
```

Check API RED:

```text
Rate      = 1,200/sec
Errors    = 0.5%
P95       = 1.8 sec
```

Then check infrastructure:

```text
API CPU       = 60%
API Memory    = 65%
MySQL CPU     = 95%
```

Check MySQL:

```text
Connections       = 950 / 1000
Slow queries      = increasing
Query latency     = 1.5 sec
```

Tracing shows:

```text
API
 │
 └── MySQL
       │
       └── Slow Query = 1.4 sec
```

Root cause:

```text
Slow MySQL query
      │
      ▼
High DB latency
      │
      ▼
API latency increases
      │
      ▼
User experiences slow API
```

This is the real value of observability:

**Metric → Investigation → Trace → Root Cause**

---

# 10.19 Production Observability Architecture

```text
                         USERS
                           │
                           ▼
                     ┌───────────┐
                     │  Ingress  │
                     └─────┬─────┘
                           │
                           ▼
                     ┌───────────┐
                     │ Flask API  │
                     └─────┬─────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
            MySQL       RabbitMQ      External
              │            │
              │            ▼
              │       Celery Workers
              │
              ▼
         Read Replicas


        ┌─────────────────────────────────┐
        │          TELEMETRY              │
        ├─────────────────────────────────┤
        │ Metrics → Prometheus             │
        │ Logs    → Loki                   │
        │ Traces  → OpenTelemetry          │
        │ Azure   → Azure Monitor           │
        └───────────────┬─────────────────┘
                        │
                        ▼
                   ┌──────────┐
                   │ Grafana  │
                   └────┬─────┘
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
          Dashboards            Alerts
                                  │
                                  ▼
                         Notification / On-call
```

---

# 10.20 Observability Strategy by Component

| Component       | Metrics       | Logs          | Traces   | Alerts |
| --------------- | ------------- | ------------- | -------- | ------ |
| Flask API       | ✅             | ✅             | ✅        | ✅      |
| Celery Worker   | ✅             | ✅             | ✅        | ✅      |
| Celery Beat     | ✅             | ✅             | Optional | ✅      |
| RabbitMQ        | ✅             | ✅             | Optional | ✅      |
| MySQL           | ✅             | ✅             | ✅        | ✅      |
| AKS Nodes       | ✅             | ✅             | —        | ✅      |
| Kubernetes Pods | ✅             | ✅             | —        | ✅      |
| Azure Resources | Azure Monitor | Azure Monitor | —        | ✅      |

---

# 10.21 Observability Design Principles

### 1. Monitor business impact

Do not monitor only:

```text
CPU
Memory
Disk
```

Also monitor:

```text
Request success
Request latency
Job success
Queue processing time
Availability
```

---

### 2. Alert on symptoms, not every cause

Prefer:

```text
API availability below SLO
```

over dozens of unrelated low-level alerts.

---

### 3. Use correlation IDs

A request should be traceable across services:

```text
Request ID
Trace ID
```

Example:

```text
User Request
     │
     ▼
Flask API
     │ trace_id=abc123
     ▼
RabbitMQ
     │ trace_id=abc123
     ▼
Celery Worker
     │ trace_id=abc123
     ▼
MySQL
```

This dramatically reduces troubleshooting time.

---

### 4. Design dashboards around questions

Instead of:

> "Show me every metric."

build dashboards that answer:

```text
Is the API healthy?

Why is the API slow?

Are workers keeping up?

Is RabbitMQ building backlog?

Is MySQL approaching capacity?

Are we meeting our SLO?
```

---

### 5. Observability must be actionable

Every critical alert should have:

```text
Alert
  │
  ├── What happened?
  ├── Why does it matter?
  ├── What should I check?
  └── What should I do?
```

---

# 10.22 Recommended SLOs

Example starting point:

| Service   | SLI                |         Example SLO |
| --------- | ------------------ | ------------------: |
| Flask API | Availability       |               99.9% |
| Flask API | P95 latency        |            < 300 ms |
| Flask API | Error rate         |                < 1% |
| Celery    | Successful jobs    |               99.5% |
| RabbitMQ  | Message processing | 99.9% within target |
| MySQL     | Availability       |               99.9% |

These are **example targets**, not universal requirements. Actual SLOs should be based on business requirements and user expectations.

---

# 10.23 Monitoring vs Observability

| Monitoring             | Observability                      |
| ---------------------- | ---------------------------------- |
| Detects known problems | Helps investigate unknown problems |
| Dashboards             | Metrics + Logs + Traces            |
| Alerts                 | Correlated evidence                |
| "Something is wrong"   | "Why is it wrong?"                 |
| Threshold oriented     | Context oriented                   |

Both are required.

---

# 10.24 Chapter Summary

The production observability stack is:

```text
Prometheus
    │
    └── Metrics

Grafana
    │
    └── Dashboards

Loki
    │
    └── Logs

OpenTelemetry
    │
    └── Distributed Traces

Azure Monitor
    │
    └── Azure Resource Monitoring
```

The operational framework is:

```text
Metrics
   │
   ▼
Golden Signals
   │
   ├── Latency
   ├── Traffic
   ├── Errors
   └── Saturation
   │
   ▼
RED / USE
   │
   ▼
SLIs
   │
   ▼
SLOs
   │
   ▼
Error Budget
   │
   ▼
Engineering Decisions
```

### Key Takeaways

* **Prometheus** collects metrics.
* **Grafana** visualizes telemetry.
* **Loki** aggregates logs.
* **OpenTelemetry** provides distributed tracing.
* **Azure Monitor** monitors Azure resources.
* **Golden Signals** provide a high-level view of service health.
* **RED** is useful for applications and APIs.
* **USE** is useful for infrastructure and resources.
* **SLIs** measure actual service behavior.
* **SLOs** define the desired reliability.
* **Error budgets** turn reliability into an engineering decision.
* Good observability should help engineers move from **symptom → evidence → root cause → action**.

---

## Next Chapter

**Part 11 – Security Architecture**

Next we will cover:

* Kubernetes security
* Azure identity and Managed Identity
* Secrets management
* Network security
* RBAC
* Pod security
* Container security
* Database security
* RabbitMQ security
* TLS
* Security monitoring
* Production security checklist
