# Enterprise Cloud-Native Architecture Handbook

# Part 4 – Kubernetes & AKS Architecture

> **Objective**
>
> This chapter defines how the application is deployed on **Azure Kubernetes Service (AKS)**. It covers cluster design, node pools, networking, scheduling, storage, security, autoscaling, and production deployment practices. The goal is to build a Kubernetes platform that is scalable, resilient, and operationally efficient.

---

# 4.1 AKS Cluster Overview

AKS is the **control plane** that manages all application workloads.

Rather than deploying directly onto virtual machines, applications are packaged as containers and orchestrated by Kubernetes.

High-level architecture:

```text
                        Azure Subscription
                               │
                               ▼
                    Azure Kubernetes Service
                               │
         ┌─────────────────────┼─────────────────────┐
         │                     │                     │
    System Node Pool      API Node Pool      Worker Node Pool
         │                     │                     │
    Core Components        Flask API Pods     Celery Workers
         │
         ├── CoreDNS
         ├── Metrics Server
         ├── Ingress Controller
         ├── Prometheus
         └── Grafana
```

---

# 4.2 Why Multiple Node Pools?

A common beginner mistake is running everything on one node pool.

Instead of:

```text
Node Pool

Flask
RabbitMQ
Monitoring
Ingress
Celery
```

Use dedicated pools.

Recommended architecture:

```text
AKS Cluster

├── System Pool
├── API Pool
├── Worker Pool
├── Messaging Pool
```

Benefits:

* Independent scaling
* Better resource utilization
* Easier upgrades
* Isolation of noisy workloads
* Improved fault tolerance

---

# 4.3 Recommended Node Pool Design

## System Node Pool

Purpose:

Runs Kubernetes platform components.

Typical workloads:

* CoreDNS
* Metrics Server
* Ingress Controller
* Prometheus
* Grafana
* Loki
* OpenTelemetry Collector

Characteristics:

* Small
* Stable
* Low scaling frequency

Example:

| Property  | Value  |
| --------- | ------ |
| VM Size   | D4s_v5 |
| Min Nodes | 2      |
| Max Nodes | 3      |

---

## API Node Pool

Purpose:

Runs Flask API.

Characteristics:

* CPU intensive
* Stateless
* HPA enabled

Example:

| Property  | Value  |
| --------- | ------ |
| VM Size   | D8s_v5 |
| Min Nodes | 3      |
| Max Nodes | 10     |

---

## Worker Node Pool

Purpose:

Runs Celery workers.

Characteristics:

* CPU intensive
* Memory intensive
* Highly elastic

Example:

| Property  | Value   |
| --------- | ------- |
| VM Size   | D16s_v5 |
| Min Nodes | 0–2     |
| Max Nodes | 20      |

---

## Messaging Node Pool

Purpose:

Runs RabbitMQ.

Characteristics:

* Stateful
* Persistent disks
* High availability

Example:

| Property | Value  |
| -------- | ------ |
| VM Size  | D8s_v5 |
| Nodes    | 3      |

---

# 4.4 Namespace Strategy

Avoid deploying everything into the `default` namespace.

Recommended structure:

```text
aks-cluster

├── ingress-system
├── monitoring
├── messaging
├── production
├── staging
├── development
```

Benefits:

* Resource isolation
* RBAC separation
* Easier monitoring
* Cleaner deployments

---

# 4.5 Production Deployment Layout

```text
production namespace

├── Flask Deployment
├── Flask Service
├── HPA
├── ConfigMap
├── Secret

├── Celery Deployment
├── KEDA ScaledObject

├── Celery Beat Deployment

├── NetworkPolicy

├── PodDisruptionBudget

├── ServiceAccount
```

---

# 4.6 Kubernetes Resource Relationships

```text
Deployment
      │
ReplicaSet
      │
Pods
      │
Containers
```

Services expose deployments.

Ingress exposes services.

---

# 4.7 Networking Architecture

```text
Internet
      │
Azure Front Door
      │
Application Gateway
      │
NGINX Ingress
      │
ClusterIP Service
      │
Flask Pods
```

Internal communication:

```text
Flask

↓

RabbitMQ Service

↓

RabbitMQ Pods
```

```text
Flask

↓

MySQL

(Private Endpoint)
```

---

# 4.8 Service Types

| Service      | Usage                         |
| ------------ | ----------------------------- |
| ClusterIP    | Internal communication        |
| LoadBalancer | External exposure (if needed) |
| NodePort     | Development/testing           |
| ExternalName | External service mapping      |

Recommendation:

Use **ClusterIP** for all internal services.

---

# 4.9 Ingress Architecture

```text
Internet

↓

Azure Front Door

↓

Application Gateway

↓

NGINX Ingress

↓

Flask Service

↓

Pods
```

Ingress responsibilities:

* SSL termination
* Routing
* Rate limiting
* Path-based routing
* Host-based routing

---

# 4.10 Scheduling Strategy

Pods should not all run on the same node.

Example:

❌ Bad

```text
Node 1

Flask-1
Flask-2
Flask-3
Flask-4
```

If Node 1 fails:

Entire API becomes unavailable.

---

Preferred:

```text
Node 1

Flask-1

----------------

Node 2

Flask-2

----------------

Node 3

Flask-3

----------------

Node 4

Flask-4
```

Use:

* Pod anti-affinity
* Topology spread constraints

---

# 4.11 RabbitMQ Scheduling

RabbitMQ pods should never share the same node.

Example:

```text
Node-1

RabbitMQ-1

-----------------

Node-2

RabbitMQ-2

-----------------

Node-3

RabbitMQ-3
```

This prevents a single node failure from taking down the messaging cluster.

---

# 4.12 Resource Requests & Limits

Every workload should define requests and limits.

Example:

| Component | CPU Request | CPU Limit | Memory Request | Memory Limit |
| --------- | ----------- | --------- | -------------- | ------------ |
| Flask     | 500m        | 1 CPU     | 512Mi          | 1Gi          |
| Worker    | 500m        | 2 CPU     | 1Gi            | 2Gi          |
| RabbitMQ  | 2 CPU       | 4 CPU     | 4Gi            | 8Gi          |

Benefits:

* Better scheduling
* Prevent noisy neighbors
* More predictable autoscaling

---

# 4.13 Autoscaling Architecture

```text
Incoming Traffic

↓

Flask Pods

↓

CPU

↓

HPA

↓

More Pods

↓

Pending Pods

↓

Cluster Autoscaler

↓

More Nodes
```

Worker scaling:

```text
Queue Length

↓

RabbitMQ

↓

KEDA

↓

Worker Pods

↓

Cluster Autoscaler

↓

New Nodes
```

---

# 4.14 Storage Architecture

Stateless components:

* Flask
* Celery

No persistent volumes.

Stateful components:

RabbitMQ

```text
PVC

↓

Azure Managed Disk
```

Database:

Azure MySQL Flexible Server (managed outside the cluster).

---

# 4.15 Secrets Management

Avoid storing secrets in container images or Git repositories.

Recommended flow:

```text
Azure Key Vault

↓

CSI Driver

↓

Kubernetes Secret

↓

Pod
```

Examples of secrets:

* Database password
* RabbitMQ credentials
* API keys
* TLS certificates

---

# 4.16 Configuration Management

Use ConfigMaps for non-sensitive configuration.

Examples:

* Log level
* Feature flags
* API URLs
* Queue names

Keep secrets separate from configuration.

---

# 4.17 High Availability

Design for failures.

Example:

```text
Node Failure

↓

Pods Lost

↓

Deployment Controller

↓

Pods Scheduled

↓

Healthy Node
```

For RabbitMQ:

```text
RabbitMQ Node Failure

↓

Quorum Maintained

↓

Cluster Continues
```

---

# 4.18 Rolling Updates

Deployment strategy:

```text
Version 1

↓

Create Version 2 Pods

↓

Readiness Check

↓

Route Traffic

↓

Remove Version 1
```

Benefits:

* Zero downtime
* Easy rollback
* Safe deployments

---

# 4.19 PodDisruptionBudget (PDB)

Protect critical workloads during voluntary disruptions.

Recommended examples:

| Component  | Min Available |
| ---------- | ------------- |
| Flask      | 80%           |
| RabbitMQ   | 2 of 3        |
| Monitoring | 1             |

---

# 4.20 Health Probes

Every production pod should implement:

### Startup Probe

Determines when the application has finished starting.

### Readiness Probe

Determines whether the pod is ready to receive traffic.

### Liveness Probe

Detects hung or unhealthy applications and restarts them.

Without these probes, Kubernetes cannot accurately manage application health.

---

# 4.21 Security Architecture

Recommendations:

* Enable RBAC.
* Use NetworkPolicies to restrict pod-to-pod communication.
* Run containers as non-root users.
* Use read-only root filesystems where possible.
* Scan container images for vulnerabilities.
* Apply Pod Security Standards.
* Restrict egress traffic to required destinations.

---

# 4.22 Recommended Production Topology

```text
                           Internet
                               │
                    Azure Front Door
                               │
                 Azure Application Gateway
                               │
                    NGINX Ingress Controller
                               │
                 +-------------+-------------+
                 |                           |
          Flask API Service          Monitoring UI
                 │
        +--------+--------+
        |                 |
     Flask Pods (HPA)     |
        │                 |
        +--------+--------+
                 │
        RabbitMQ Cluster (3 Nodes)
                 │
       +---------+---------+
       |                   |
 Celery Beat         Celery Workers (KEDA)
                           │
                           ▼
             Azure MySQL Flexible Server
                   │
        +----------+----------+
        |                     |
     Primary            Read Replicas
```

---

# 4.23 Production Readiness Checklist

Before going live, verify:

* ✔ Separate node pools for system, API, workers, and messaging.
* ✔ Namespaces are defined for isolation.
* ✔ Resource requests and limits are configured.
* ✔ HPA and KEDA limits are validated.
* ✔ Cluster Autoscaler is enabled.
* ✔ Pod anti-affinity and topology spread constraints are applied.
* ✔ RabbitMQ uses quorum queues with persistent storage.
* ✔ Secrets are sourced securely (e.g., Azure Key Vault).
* ✔ Readiness, liveness, and startup probes are configured.
* ✔ PDBs protect critical workloads.
* ✔ Monitoring, logging, and alerting are operational.
* ✔ Rolling update strategy has been tested.

---

# Chapter Summary

A Kubernetes platform should be designed as a **layered infrastructure**, not simply as a place to run containers.

This chapter established the core infrastructure principles:

* **Dedicated node pools** isolate workloads and allow independent scaling.
* **Namespaces** provide organizational and security boundaries.
* **Deployments, Services, and Ingress** form the application's runtime topology.
* **HPA, KEDA, and Cluster Autoscaler** work together to scale applications and infrastructure.
* **Scheduling policies, PDBs, and health probes** improve resilience and availability.
* **Secrets management and network controls** strengthen the security posture.

A well-designed AKS foundation enables the application layer to scale reliably while minimizing operational complexity.

---

# Next Chapter

**Part 5 – Scaling Strategy (HPA, KEDA & Cluster Autoscaler)** will focus exclusively on autoscaling. We'll explore how these three mechanisms interact, derive scaling formulas, tune thresholds, avoid oscillation ("flapping"), prevent database overload, and validate scaling behavior through load testing and production metrics. This chapter will connect the capacity planning from Part 2 with the infrastructure design from Part 4 into a cohesive, production-ready scaling strategy.
