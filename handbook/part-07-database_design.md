# Part 7 – Database Design

## Overview

The database is one of the most important components in a highly scalable application because it is stateful and cannot scale in exactly the same way as stateless API and worker pods.

The reference architecture uses **Azure Database for MySQL Flexible Server** with:

- One primary database for writes
- Multiple read replicas for read scaling
- Connection pooling
- Query and index optimization
- Replication-lag monitoring
- Backup and recovery
- Optional caching
- Partitioning and sharding as future scaling strategies

The goal is to scale database workloads without sacrificing data consistency, reliability, or operational simplicity.

---

# 7.1 Database Architecture

```text
                         Application
                              │
                ┌─────────────┴─────────────┐
                │                           │
             Writes                       Reads
                │                           │
                ▼                           ▼
        MySQL Primary              Read Routing Layer
                │                           │
                │                 ┌─────────┼─────────┐
                │                 │         │         │
                │                 ▼         ▼         ▼
                │               Replica1 Replica2 ... Replica10
                │
                ▼
        Asynchronous Replication
```

### Key Design

| Component | Strategy |
|-----------|----------|
| Writes | MySQL Primary |
| Reads | Read Replicas |
| Read Replicas | Up to 10 based on workload |
| HA | Azure Flexible Server HA |
| Connection Management | Application Connection Pool |
| Caching | Optional |
| Partitioning | Based on data growth |
| Sharding | Future roadmap |

Azure Database for MySQL Flexible Server supports up to 10 read replicas. Read replicas use asynchronous replication and are intended for read scaling rather than automatic high availability failover. :contentReference[oaicite:1]{index=1}

---

# 7.2 Primary and Read Replica Strategy

The primary database handles:

- INSERT
- UPDATE
- DELETE
- Transactions requiring strong consistency

Read replicas handle:

- SELECT queries
- Reporting
- Search/read-heavy workloads
- Analytics queries that can tolerate replication delay

```text
                 Flask API
                    │
          ┌─────────┴─────────┐
          │                   │
       Write Path          Read Path
          │                   │
          ▼                   ▼
      Primary DB        Read Router
                              │
                 ┌────────────┼────────────┐
                 ▼            ▼            ▼
              Replica 1    Replica 2   Replica N
```

### Important

Do not send every read to replicas automatically.

Reads that require the latest committed data may need to go to the primary.

---

# 7.3 Read Consistency

Read replicas use asynchronous replication.

Therefore:

```text
Primary Commit
      │
      ▼
Replication
      │
      ▼
Read Replica
```

There can be a delay between the primary and replica.

This creates two important consistency models.

| Model | Description | Example |
|-------|-------------|---------|
| Strong Consistency | Read the latest committed value | Payment status |
| Eventual Consistency | Replica may temporarily lag | Product catalog |

### Example

A user creates an order:

```text
POST /orders

      ↓

Primary DB

      ↓

Order Created
```

Immediately reading from a replica may temporarily return:

```text
Order Not Found
```

because replication has not completed.

### Recommendation

Use the primary for:

- Immediately-after-write reads
- Financial transactions
- Critical state transitions
- Data requiring strong consistency

Use replicas for:

- Catalog browsing
- Reporting
- Dashboards
- Search
- Non-critical read workloads

---

# 7.4 Replication Lag Monitoring

Replication lag is a critical metric when using read replicas.

```text
Primary

   │
   │ Transactions
   ▼

Replication

   │
   │ Delay
   ▼

Replica
```

Monitor:

- Replication lag
- Replica availability
- CPU
- Memory
- Storage
- Query latency

Azure Monitor exposes a **replication lag in seconds** metric for MySQL Flexible Server read replicas. :contentReference[oaicite:2]{index=2}

### Example Policy

```text
Replication Lag

< 5 sec
    │
    ▼
Healthy

5–30 sec
    │
    ▼
Warning

> 30 sec
    │
    ▼
Critical
```

The actual thresholds should be determined by application consistency requirements.

---

# 7.5 Read Replica Failover

Read replicas should not be confused with Azure Flexible Server HA.

```text
Azure HA

Primary
   │
   ▼
Passive Standby

Purpose:
High Availability
```

versus:

```text
Read Replicas

Primary
   │
   ├── Replica 1
   ├── Replica 2
   └── Replica N

Purpose:
Read Scaling
```

Azure's HA standby is passive and isn't available for normal read/write application traffic. Read replicas are separate servers intended for scaling read workloads. :contentReference[oaicite:3]{index=3}

There is no automatic failover between an Azure MySQL source server and its read replicas. A replica must be promoted manually if it is chosen as a new writable server. :contentReference[oaicite:4]{index=4}

---

# 7.6 Transaction Design

Transactions should be kept:

- Short
- Focused
- Predictable

Avoid:

```text
BEGIN TRANSACTION

Long business processing

External API call

Large computation

Multiple unrelated queries

COMMIT
```

Prefer:

```text
Validate

↓

Perform required DB operations

↓

Commit

↓

Perform asynchronous work
```

### Key Principles

- Keep transaction scope small.
- Avoid unnecessary locks.
- Avoid long-running transactions.
- Use appropriate isolation levels.
- Make retries safe through idempotent operations.

---

# 7.7 Index Strategy

Indexes improve query performance by allowing MySQL to locate matching rows without scanning the entire table. However, unnecessary indexes increase storage and write overhead because indexes must also be maintained during INSERT, UPDATE, and DELETE operations. :contentReference[oaicite:5]{index=5}

### Common Index Types

| Index | Purpose |
|-------|---------|
| Primary Key | Unique row identification |
| Unique Index | Enforce uniqueness |
| Single-column Index | Optimize common lookups |
| Composite Index | Optimize multi-column queries |
| Covering Index | Reduce table lookups |

### Example

Query:

```sql
SELECT *
FROM orders
WHERE customer_id = ?
  AND status = ?;
```

Potential index:

```sql
INDEX(customer_id, status)
```

Index design should always be driven by actual query patterns.

---

# 7.8 Query Optimization

Poor queries can become a database bottleneck even when sufficient CPU and memory are available.

### Optimization Process

```text
Slow Query
    │
    ▼
Identify Query
    │
    ▼
EXPLAIN / Query Analysis
    │
    ▼
Check Index Usage
    │
    ▼
Optimize Query / Index
    │
    ▼
Measure Again
```

Look for:

- Full table scans
- Missing indexes
- Excessive joins
- Large result sets
- Unnecessary columns
- N+1 query patterns
- Long-running transactions

---

# 7.9 Slow Query Management

Monitor slow queries using:

- MySQL slow query logging
- Azure Monitor
- Application metrics
- Query execution analysis

Important metrics include:

| Metric | Purpose |
|--------|---------|
| Query Latency | Detect slow operations |
| Queries/sec | Measure database workload |
| CPU | Detect compute saturation |
| Connections | Detect connection pressure |
| Storage I/O | Detect I/O bottlenecks |
| Lock Waits | Detect contention |

### Remediation Flow

```text
Slow Query Detected

↓

Identify Query Pattern

↓

Analyze Execution Plan

↓

Optimize SQL / Index

↓

Load Test

↓

Deploy

↓

Monitor
```

---

# 7.10 Connection Pool Management

Every API and worker pod may maintain database connections.

Therefore:

```text
Total Connections

=

API Pods × API Pool

+

Worker Pods × Worker Pool

+

Admin

+

Monitoring
```

Example:

```text
20 API Pods × 20

+

20 Worker Pods × 10

+

30 Other

=

630 Connections
```

Connection pools should be sized together with the database's maximum connection capacity.

### Key Principles

- Avoid unnecessarily large pools.
- Reuse connections.
- Configure connection timeouts.
- Configure idle connection handling.
- Leave operational headroom.
- Monitor connection utilization.

Connection pooling is especially important when HPA and KEDA increase pod counts because every new pod can create additional database connections.

---

# 7.11 Partitioning Strategy

Partitioning divides a large logical table into smaller physical partitions.

Example:

```text
Orders

├── 2026_Q1
├── 2026_Q2
├── 2026_Q3
└── 2026_Q4
```

Potential partition keys include:

- Date
- Tenant
- Region
- Business domain

Partitioning can help when tables become very large and queries naturally target specific partitions.

### Use Partitioning When

- Tables contain very large datasets.
- Queries frequently filter by the partition key.
- Data has natural lifecycle boundaries.

Do not introduce partitioning simply because a table is growing. Validate the query patterns and operational benefits first.

---

# 7.12 Sharding – Future Roadmap

Sharding distributes data across multiple database instances.

```text
                Application
                     │
               Shard Router
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
     Shard 1      Shard 2      Shard 3
```

Example:

```text
Customer ID

1–1M       → Shard 1
1M–2M      → Shard 2
2M–3M      → Shard 3
```

### Sharding Benefits

- Horizontal database scaling
- Larger aggregate storage capacity
- Workload isolation

### Sharding Challenges

- Cross-shard queries
- Transactions
- Rebalancing
- Operational complexity
- Data distribution
- Application complexity

### Recommendation

Treat sharding as a **future scaling strategy**, not the default architecture.

First optimize:

```text
Query Performance
        ↓
Indexes
        ↓
Connection Pools
        ↓
Read Replicas
        ↓
Partitioning
        ↓
Sharding
```

---

# 7.13 Caching Strategy

Caching can reduce database load and improve application latency.

```text
Client

↓

Flask API

↓

Cache

├── HIT → Return Data
│
└── MISS
       ↓
    MySQL
       ↓
    Cache
       ↓
    Response
```

Common candidates:

- Frequently accessed data
- Product/catalog information
- Configuration
- Session-related data
- Expensive read operations

---

# 7.14 Cache Invalidation

Caching introduces a new consistency problem:

> **How do we know when cached data is stale?**

Common strategies:

| Strategy | Description |
|----------|-------------|
| TTL | Cache expires automatically |
| Explicit Invalidation | Delete cache after update |
| Write-Through | Update cache and database together |
| Cache-Aside | Application manages cache reads/writes |

### Recommended Default

For many read-heavy APIs:

```text
Cache-Aside + TTL
```

Flow:

```text
Application

↓

Check Cache

├── HIT → Return
│
└── MISS
      ↓
    MySQL
      ↓
    Update Cache
      ↓
    Return
```

Cache should not become the system of record.

---

# 7.15 Backup & Recovery

Database reliability requires both high availability and recoverability.

```text
MySQL Flexible Server

        │
        ├── High Availability
        │
        ├── Automated Backups
        │
        └── Point-in-Time Recovery
```

Important distinction:

```text
HA

→ Minimize downtime

Backup

→ Recover data
```

A highly available database does not replace backups.

Azure Database for MySQL Flexible Server provides managed backup capabilities, while HA provides automatic failover between the primary and standby architecture. :contentReference[oaicite:6]{index=6}

---

# 7.16 Database Monitoring

The database should be monitored as a first-class production component.

### Key Metrics

| Category | Metrics |
|----------|---------|
| Performance | Query latency, Queries/sec |
| Compute | CPU, Memory |
| Connections | Active Connections, Connection Utilization |
| Replication | Replication Lag |
| Storage | Storage Used, I/O |
| Reliability | Availability, Failovers |
| Backup | Backup status, Backup storage |

---

# 7.17 Production Database Architecture

```text
                         AKS
                          │
              ┌───────────┴───────────┐
              │                       │
          Flask API              Celery Workers
              │                       │
              └───────────┬───────────┘
                          │
                  Connection Pools
                          │
             ┌────────────┴────────────┐
             │                         │
          Writes                     Reads
             │                         │
             ▼                         ▼
      MySQL Primary              Read Router
             │                         │
             │              ┌──────────┼──────────┐
             │              ▼          ▼          ▼
             │          Replica 1  Replica 2   Replica N
             │
             ▼
       Azure HA Standby
             │
             ▼
       Backup / Recovery
```

---

# 7.18 Database Design Principles

| Principle | Design Decision |
|-----------|-----------------|
| Write Scaling | Primary database |
| Read Scaling | Read replicas |
| High Availability | Azure Flexible Server HA |
| Consistency | Primary for critical reads |
| Query Performance | Index and query optimization |
| Connection Management | Application pooling |
| Large Tables | Partitioning where justified |
| Extreme Scale | Sharding as future option |
| Read Optimization | Cache where appropriate |
| Recovery | Automated backup + restore |
| Monitoring | Query, connection, replication and resource metrics |

---

# Chapter Summary

The database architecture combines **write centralization, read scaling, connection management, query optimization, caching, partitioning, and high availability**.

The core strategy is:

```text
Optimize Queries
      ↓
Optimize Indexes
      ↓
Manage Connections
      ↓
Scale Reads
      ↓
Partition Large Data
      ↓
Introduce Caching
      ↓
Consider Sharding
```

The goal is not to introduce database complexity early. Instead, scale the database progressively as workload and data volume increase.

---

## Next Chapter

➡️ **Part 8 – Queue Architecture**