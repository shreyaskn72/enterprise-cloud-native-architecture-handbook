# Part 3 – Component Deep Dive

## Overview

This chapter introduces the core components of the reference architecture. Each component plays a specific role in delivering a scalable, resilient, and cloud-native platform. Understanding these responsibilities provides the foundation for the detailed implementation, scaling, monitoring, security, and operational topics covered in later chapters.

The components discussed in this chapter are:

- Flask API
- RabbitMQ
- Celery Workers
- Celery Beat
- Azure Database for MySQL Flexible Server

---

# 3.1 Flask API

## Purpose

The Flask API is the primary entry point for client requests. It exposes REST APIs, executes business logic, performs synchronous database operations, and publishes long-running tasks to RabbitMQ for asynchronous processing.

## Architecture

```text
Client
   │
   ▼
Flask API
   ├── Authentication
   ├── Validation
   ├── Business Logic
   ├── Azure MySQL
   └── RabbitMQ
```

## Deployment

| Property | Design |
|----------|--------|
| Kubernetes Resource | Deployment |
| Service | ClusterIP |
| Scaling | Horizontal Pod Autoscaler (HPA) |
| Design | Stateless |

## Key Responsibilities

- Handle HTTP requests
- Validate input
- Execute business logic
- Read and write application data
- Publish background tasks
- Return API responses

## Key Design Considerations

| Topic | Summary |
|-------|---------|
| Responsibilities & Request Handling | Handle authentication, validation, business logic, synchronous database operations, and asynchronous task publishing. |
| Horizontal & Vertical Scaling | Scale horizontally using HPA and vertically by adjusting CPU and memory resources when required. |
| Resource Requests & Limits | Configure requests for scheduling and limits to prevent resource exhaustion. |
| Health Probes | Implement startup, readiness, and liveness probes for high availability and zero-downtime deployments. |
| Connection Pooling | Reuse database connections efficiently and size pools based on expected concurrency. |
| Failure Handling | Implement retries, graceful shutdown, request timeouts, and Kubernetes self-healing. |
| Security | Apply TLS, authentication, authorization, secure secrets management, and least-privilege access. |
| Metrics & Instrumentation | Expose application metrics, structured logs, and distributed traces for observability. |

---

# 3.2 RabbitMQ

## Purpose

RabbitMQ provides reliable asynchronous messaging between producers and consumers. It decouples request processing from background task execution, allowing the platform to process long-running operations efficiently.

## Architecture

```text
Flask API
     │
     ▼
 RabbitMQ Exchange
     │
     ▼
 RabbitMQ Queue
     │
     ▼
 Celery Workers
```

## Deployment

| Property | Design |
|----------|--------|
| Kubernetes Resource | StatefulSet |
| Cluster | Three Nodes |
| Queue Type | Quorum Queue |
| Storage | Persistent Volumes |

## Key Responsibilities

- Route messages
- Store messages reliably
- Deliver tasks to workers
- Support retries and dead-letter queues
- Enable asynchronous processing

## Key Design Considerations

| Topic | Summary |
|-------|---------|
| Exchange Types | Use Direct, Topic, Fanout, or Headers exchanges based on routing requirements. |
| Queue Design | Create durable queues with appropriate naming, routing, and retention policies. |
| Bindings | Configure exchange-to-queue bindings for flexible message routing. |
| Publisher Patterns | Use publisher confirms and persistent messages to improve delivery guarantees. |
| Consumer Strategy | Optimize consumer concurrency and acknowledgement strategy for reliability and throughput. |
| Dead Letter Queue (DLQ) | Route failed messages for inspection and recovery. |
| Retry Queue | Implement delayed retries with exponential backoff for transient failures. |
| Quorum Queues | Use quorum queues to achieve high availability and fault tolerance. |
| Persistence & Durability | Persist messages to disk to prevent data loss during failures. |
| Cluster Architecture | Deploy a three-node RabbitMQ cluster to maintain quorum and resilience. |

---

# 3.3 Celery Workers

## Purpose

Celery Workers execute asynchronous tasks received from RabbitMQ. Background processing improves application responsiveness by moving long-running operations outside the request-response cycle.

## Architecture

```text
RabbitMQ

↓

Celery Workers

↓

Azure MySQL
```

## Deployment

| Property | Design |
|----------|--------|
| Kubernetes Resource | Deployment |
| Scaling | KEDA |
| Design | Stateless |

## Key Responsibilities

- Consume background tasks
- Execute business logic
- Update application data
- Retry failed tasks

## Key Design Considerations

| Topic | Summary |
|-------|---------|
| Worker Lifecycle | Workers transition through startup, task execution, idle, and graceful shutdown states. |
| Horizontal Autoscaling | Scale automatically using KEDA based on RabbitMQ queue length. |
| Task Acknowledgements | Configure acknowledgement modes to balance throughput and reliability. |
| Retry Strategy | Apply retries with exponential backoff for transient failures. |
| Idempotency | Ensure tasks can be executed multiple times safely. |
| Task Scheduling | Execute asynchronous jobs published by APIs or Celery Beat. |
| Concurrency | Tune worker concurrency according to CPU, memory, and workload characteristics. |
| Prefetch Limits | Configure prefetch count to optimize fairness and throughput. |
| Memory Management | Monitor memory usage and recycle workers periodically to prevent leaks. |

---

# 3.4 Celery Beat

## Purpose

Celery Beat schedules recurring tasks and publishes them to RabbitMQ. It enables automated execution of periodic jobs without user interaction.

## Architecture

```text
Scheduler

↓

Celery Beat

↓

RabbitMQ

↓

Celery Workers
```

## Deployment

| Property | Design |
|----------|--------|
| Kubernetes Resource | Deployment |
| Replicas | Single Replica |
| Scaling | Not Required |

## Key Responsibilities

- Schedule recurring tasks
- Publish scheduled jobs
- Trigger automated workflows

## Key Design Considerations

| Topic | Summary |
|-------|---------|
| Scheduling | Support cron-based and interval-based task execution. |
| Single Replica | Deploy one active scheduler to prevent duplicate task execution. |
| Integration | Publish scheduled tasks to RabbitMQ for worker consumption. |
| Monitoring | Track missed schedules and task publishing failures. |

---

# 3.5 Azure Database for MySQL Flexible Server

## Purpose

Azure Database for MySQL Flexible Server provides managed relational storage for the platform. It supports transactional workloads, automated backups, read replicas, and managed high availability.

## Architecture

```text
Writes

↓

Primary

↓

Replication

↓

Read Replicas

↓

Read Requests
```

## Deployment

| Property | Design |
|----------|--------|
| Platform | Azure Managed Service |
| Primary | One Instance |
| Read Replicas | Multiple Replicas |
| Backups | Automated |

## Key Responsibilities

- Store application data
- Process transactional writes
- Serve read requests
- Replicate data
- Provide backup and recovery

## Key Design Considerations

| Topic | Summary |
|-------|---------|
| Primary–Replica Architecture | Direct writes to the primary instance while distributing read traffic across replicas. |
| Read Routing | Route read-only queries to replicas while monitoring replication lag. |
| Replication Lag | Monitor lag to maintain read consistency and application performance. |
| Connection Pool Management | Size connection pools within the database's maximum connection limit. |
| Backup Strategy | Enable automated backups and validate restore procedures regularly. |
| Failover | Use managed failover capabilities to improve availability. |
| Data Partitioning | Consider partitioning strategies for very large datasets and high-volume workloads. |
| Index Design & Optimization | Design efficient indexes to improve query performance and reduce latency. |

---

# 3.6 Component Comparison

| Component | Primary Responsibility | Scaling Strategy | Deployment |
|-----------|------------------------|------------------|------------|
| Flask API | Request Processing | HPA | Deployment |
| RabbitMQ | Asynchronous Messaging | Capacity-Based | StatefulSet |
| Celery Workers | Background Task Processing | KEDA | Deployment |
| Celery Beat | Task Scheduling | Single Replica | Deployment |
| Azure MySQL | Persistent Data Storage | Read Replicas | Managed Azure Service |

---

# 3.7 Component Interaction

```text
Client
   │
   ▼
Flask API
   ├──────────────► Azure MySQL (Read / Write)
   │
   └──────────────► RabbitMQ
                        ▲
                        │
                  Celery Beat
                        │
                        ▼
                 Celery Workers
                        │
                        ▼
                  Azure MySQL
```

---

# Chapter Summary

This chapter introduced the primary components of the reference architecture and explained their roles, deployment models, scaling approaches, and key design considerations. Together, these components provide a scalable, resilient, and production-ready foundation for cloud-native applications running on Azure Kubernetes Service.

Subsequent chapters explore each architectural domain in greater depth, including Kubernetes architecture, autoscaling, capacity planning, database design, messaging patterns, observability, security, disaster recovery, and performance optimization.

---

## Next Chapter

➡ **Part 4 – Kubernetes & AKS Architecture**