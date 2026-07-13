
If Part 1 answers:

> **"Why are we building this architecture?"**

then Part 2 should answer:

> **"What does the complete system look like?"**

Everything else (Kubernetes, Scaling, RabbitMQ, MySQL, Security...) should refer back to these diagrams.

For that reason, I'd make Part 2 the most visual chapter in the book.

---

# Part 2 – High-Level Architecture

> **Objective**
>
> Introduce the overall system architecture through a series of progressively detailed views. This chapter provides the architectural foundation for all subsequent chapters by explaining how the platform is organized, how components interact, and how requests, events, and data flow through the system.

---


## Overview

This chapter presents the overall architecture of the reference platform from multiple perspectives. Rather than focusing on implementation details, it illustrates how the major system components interact to deliver a scalable, resilient, and production-ready cloud-native application.

By the end of this chapter, readers will understand:

- The overall system architecture
- Logical and physical deployment models
- Azure infrastructure layout
- Kubernetes cluster organization
- Network communication paths
- Request lifecycle
- Event-driven processing
- Database read/write flow
- High availability strategy
- Key architectural decisions

---

# Architecture Views

Modern cloud-native systems are best understood through multiple architectural views. Each view focuses on a different aspect of the system, allowing engineers, architects, and operators to understand the platform from different perspectives.

| Architecture View | Purpose |
|-------------------|---------|
| Logical Architecture | Shows relationships between business components without infrastructure details |
| Physical Architecture | Shows where components are deployed |
| Azure Architecture | Shows Azure services used by the platform |
| Network Architecture | Shows external and internal communication paths |
| AKS Cluster Architecture | Shows Kubernetes node pools and workload placement |
| Request Flow | Shows synchronous request processing |
| Event Flow | Shows asynchronous and scheduled task processing |
| Data Flow | Shows database read/write operations and replication |

---

# 2.1 Logical Architecture

## Purpose

Illustrates the logical relationship between the major application components while hiding infrastructure implementation details.

```text
                           Users
                              │
                              ▼
                        Flask REST API
                     (Business Logic Layer)
                       │               │
              Synchronous         Asynchronous
                  │                    │
                  ▼                    ▼
      Azure MySQL Primary      RabbitMQ Exchange
                  │                    │
                  │                    ▼
                  │              RabbitMQ Queues
                  │                    ▲
                  │                    │
                  │             Celery Beat
                  │         (Scheduled Tasks)
                  │                    │
                  └───────────┬────────┘
                              │
                              ▼
                      Celery Workers
                              │
                              ▼
                     Azure MySQL Database
                              │
                              ▼
                        Read Replicas
```

### Key Responsibilities

| Component | Responsibility |
|-----------|----------------|
| Flask API | Request processing and business logic |
| RabbitMQ | Reliable message delivery |
| Celery Workers | Background task execution |
| Celery Beat | Time-based task scheduling |
| Azure MySQL | Persistent relational storage |

---

# 2.2 Physical Architecture

## Purpose

Illustrates where workloads execute within the production environment.

```text
                        Internet
                            │
                 Azure Front Door (Optional)
                            │
                Azure Application Gateway
                            │
                NGINX Ingress Controller
                            │
                      AKS Cluster
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
   Flask API Pods      RabbitMQ Pods     Celery Worker Pods
                                               ▲
                                               │
                                        Celery Beat Pod

Outside AKS

Azure Database for MySQL Flexible Server
```

---

# 2.3 Azure Architecture

```text
Azure Subscription

├── Azure Front Door

├── Azure Application Gateway

├── Azure Kubernetes Service

│      ├── Flask API

│      ├── RabbitMQ

│      ├── Celery Workers

│      └── Celery Beat

├── Azure Database for MySQL Flexible Server

├── Azure Key Vault

└── Azure Monitor
```

---

# 2.4 Network Architecture

```text
Internet

↓

Azure Front Door

↓

Application Gateway

↓

NGINX Ingress

↓

ClusterIP Services

↓

Pods

↓

Azure MySQL
```

### Network Layers

| Layer | Responsibility |
|--------|----------------|
| Azure Front Door | Global traffic routing |
| Application Gateway | Layer-7 load balancing and WAF |
| NGINX Ingress | Kubernetes ingress routing |
| ClusterIP Services | Internal service discovery |
| Pod Network | Pod-to-pod communication |

---

# 2.5 AKS Cluster Architecture

```text
AKS Cluster

├── System Node Pool

│      CoreDNS

│      Metrics Server

│      NGINX Ingress

│

├── API Node Pool

│      Flask API Pods

│      Horizontal Pod Autoscaler

│

├── Worker Node Pool

│      Celery Workers

│      KEDA

│

├── Messaging Node Pool

│      RabbitMQ StatefulSet

│

├── Scheduler Node Pool (Optional)

│      Celery Beat

│

└── Monitoring Node Pool

       Prometheus

       Grafana

       Loki
```

---

# 2.6 Request Flow

```text
Client

↓

Azure Front Door

↓

Application Gateway

↓

NGINX Ingress

↓

Flask API

↓

Azure MySQL

↓

HTTP Response
```

---

# 2.7 Event Flow

## Request-Driven Events

```text
Client

↓

Flask API

↓

RabbitMQ

↓

Celery Workers

↓

Azure MySQL
```

## Time-Driven Events

```text
Scheduler

↓

Celery Beat

↓

RabbitMQ

↓

Celery Workers

↓

Azure MySQL
```

---

# 2.8 Data Flow

```text
Write Request

↓

Primary Database

↓

Replication

↓

Read Replicas

↓

Read Requests
```

---

# 2.9 High Availability Design

| Layer | High Availability Strategy |
|--------|----------------------------|
| Flask API | Multiple Replicas (HPA) |
| RabbitMQ | Three-Node Quorum Cluster |
| Celery Workers | KEDA Autoscaling |
| Celery Beat | Single Active Replica |
| Azure MySQL | Managed High Availability + Read Replicas |
| AKS | Multiple Worker Nodes + Cluster Autoscaler |

---

# 2.10 Architecture Decisions

| Decision | Reason |
|----------|--------|
| Stateless APIs | Enable horizontal scaling |
| RabbitMQ | Reliable asynchronous messaging |
| Celery Workers | Background task execution |
| Celery Beat | Scheduled workload orchestration |
| Azure MySQL | Managed database operations |
| Read Replicas | Improve read scalability |
| HPA | Automatic API scaling |
| KEDA | Queue-driven worker scaling |
| Cluster Autoscaler | Automatic node provisioning |

---

# Chapter Summary

This chapter introduced the reference architecture through multiple architectural viewpoints. Together, these views establish a shared understanding of how the platform processes requests, executes background workloads, stores data, and achieves scalability and high availability.

The following chapter explores each core component individually, explaining its responsibilities, deployment model, scaling strategy, operational considerations, and production best practices.

---

# Next Chapter

➡ **Part 3 – Component Deep Dive**
