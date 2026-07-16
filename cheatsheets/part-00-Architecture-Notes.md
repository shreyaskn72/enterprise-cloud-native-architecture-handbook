# Cloud-Native Architecture — high level overview

*Written in simple language, with the "why" explained — not just the "what". Good for understanding the concept before you memorize the cheatsheet.*

---

## 1. What is this system, in plain words?

Imagine a website where:
- Users make requests (like placing an order).
- Some responses need to happen **immediately** (e.g., "show me my profile") — this is **synchronous**.
- Some tasks can happen **in the background**, later (e.g., "send a confirmation email") — this is **asynchronous**.

This architecture is built to handle **both** kinds of work, at large scale, without falling over — and to do it on **Azure Kubernetes Service (AKS)**, which is just Azure's managed way of running containers.

---

## 2. The journey of a single request

Think of it like a delivery request moving through checkpoints:

1. **User** sends a request over the internet (HTTPS).
2. **Azure Front Door / Application Gateway** — this is the "front door" of the whole system. It's the first stop, doing things like SSL termination and routing traffic to the right region.
3. **NGINX Ingress Controller** — inside the Kubernetes cluster, this decides *which service* inside the cluster should get the request.
4. **Flask API pods** — this is your actual application code, running in containers. It's "stateless," meaning no pod remembers anything between requests — this is important because it lets you freely add/remove pods without losing data.
5. From here, the request splits into two possible paths:
   - **Synchronous path**: needs an answer right now → goes straight to the **MySQL database** (primary for writes, replicas for reads).
   - **Asynchronous path**: can wait → gets dropped into **RabbitMQ** (a message queue) and picked up later by a **Celery worker**.

**Why split sync/async at all?**
Because some jobs (sending emails, generating reports, processing images) are slow. If you made the user *wait* for these, your API would feel sluggish. Instead, you say "got it, I'll do this later" and immediately respond to the user, while a background worker handles the slow part separately.

---

## 3. Meet the components (what each one actually does)

### Flask + Gunicorn (the API)
- Flask = the Python web framework that defines your routes/endpoints.
- Gunicorn = the process manager that actually runs multiple copies of Flask so it can handle multiple requests at once.
- Think of Flask as "the recipe" and Gunicorn as "the kitchen staff running multiple stoves."

### RabbitMQ (the message broker)
- A **message broker** is just a middleman that holds tasks until someone is ready to process them.
- "3-node quorum cluster" means 3 copies of RabbitMQ are running together, and they vote (quorum) to agree on what messages exist — so if one node dies, the queue doesn't lose messages.

### Celery + Celery Beat (the background workers)
- **Celery workers** = processes that pick up tasks from RabbitMQ and actually do the work (e.g., "send this email").
- **Celery Beat** = a scheduler, like a cron job. It doesn't do the work itself — it just says "hey, it's time to run the daily report task" and drops that task into RabbitMQ.
- Why only 1 replica of Beat? Because if you had 2 schedulers running, you'd get the same scheduled task fired twice. (This is a known weak spot — the handbook has an "advanced" section on making Beat highly available using a locking mechanism called Redlock.)

### Azure MySQL (the database)
- **Primary** = the one copy of the database that accepts writes (INSERT/UPDATE/DELETE).
- **Read Replicas** = copies of the database that only serve read queries (SELECT). This spreads out read traffic so the primary isn't overwhelmed.
- **Replication** = the primary continuously copies its changes to the replicas. There's a small delay in this (replication lag) — meaning replicas are always *slightly* behind the primary.

### Connection Pooling
- Every time your app talks to MySQL, it needs an open "connection" — and databases can only handle a limited number of these at once.
- Instead of opening a new connection every time (slow, wasteful), the app keeps a **pool** of pre-opened connections ready to reuse.
- **Why this matters for exams**: if you have too many API pods and workers, each holding their own pool, you can accidentally open *more connections than the database allows* — and the whole system crashes. This is why capacity planning includes a specific formula for this (see Section 5).

---

## 4. Autoscaling — teaching it like a story

Picture Black Friday traffic hitting your API:

1. **Traffic spikes.** CPU usage on your Flask pods shoots up.
2. **HPA (Horizontal Pod Autoscaler)** notices this and says "let's add more pods."
3. New pods need somewhere to run. If the existing servers (nodes) are full...
4. **Cluster Autoscaler** kicks in and adds *new virtual machines (nodes)* to the cluster so the new pods have somewhere to live.

Meanwhile, on the background-job side:

1. Thousands of async jobs pile up in RabbitMQ.
2. **KEDA** watches the queue length (not CPU!) and adds more Celery worker pods.
3. Same as before — if nodes are full, Cluster Autoscaler steps in.

**Key idea to remember**: HPA watches your *API*, KEDA watches your *queue*, and Cluster Autoscaler watches whether there's *physical room* to run everything. They work as a chain, not in isolation.

---

## 5. Why capacity planning matters (the database bottleneck)

You can scale your API pods and worker pods as much as you want — but if the database can only accept, say, 1,200 connections total, and your math says you need 1,225... **the whole system breaks**, no matter how well autoscaling works.

Simple formula to remember:
```
Total DB Connections Needed = (API pods × pool size per pod) + (Worker pods × pool size per pod) + admin/monitoring connections
```

This must always be **less than or equal to** what the database allows. That's why the handbook says: *design your max pod/worker limits based on database capacity — not the other way around.*

---

## 6. Monitoring & Observability — the "three pillars"

Students often mix these up, so here's the simple version:

| Pillar | Question it answers | Tool used here |
|---|---|---|
| **Metrics** | "How much/how many?" (CPU %, request count) | Prometheus + Grafana |
| **Logs** | "What exactly happened?" (error messages, events) | Loki |
| **Traces** | "Where did time go for this one request?" (which service was slow) | OpenTelemetry |

Together, these three let you debug a production issue instead of guessing.

---

## 7. Security — Defense in Depth (in simple terms)

"Defense in depth" means: don't rely on just one security measure — layer multiple, so if one fails, others still protect you.

- **Who can access what** → RBAC (Role-Based Access Control), at both the Azure level and Kubernetes level.
- **Where are secrets stored** → Azure Key Vault (passwords, API keys never hardcoded).
- **Network-level protection** → TLS encryption + Network Policies (which pods are even allowed to talk to which other pods).
- **Container-level protection** → scanning container images for known vulnerabilities before deployment.

---

## 8. Disaster Recovery — two acronyms to know cold

- **RTO (Recovery Time Objective)** = "How long can we afford to be down?"
- **RPO (Recovery Point Objective)** = "How much data can we afford to lose?"

Example: If your RPO is 5 minutes, your backups/replication must ensure you never lose more than 5 minutes of data.

**PITR (Point-in-Time Restore)** = the ability to restore your database to any specific moment (e.g., "restore to exactly 2:14 PM yesterday, right before the bad deployment").

---

## 9. Advanced Patterns — just the intuition (don't need deep detail for basics)

| Pattern | One-line intuition |
|---|---|
| **Circuit Breaker** | Like an electrical fuse — if a downstream service keeps failing, stop calling it for a while instead of hammering it |
| **Saga Pattern** | For transactions spanning multiple services — instead of one big transaction, do a sequence of smaller steps with "undo" actions if something fails |
| **CQRS** | Split how you *write* data from how you *read* data — they can use different models optimized for each job |
| **Rate Limiting** | Cap how many requests a user/client can make, so no one can overwhelm the system |
| **Cache Coherency** | When you have multiple caches, how do you make sure they all agree on the current data? |

---

## 10. Quick self-test questions (try to answer without looking back)

1. Why does the Flask API need to be "stateless"?
2. What's the difference between what HPA scales vs what KEDA scales?
3. Why can you not just set autoscaling max replicas to a huge number?
4. Why is there only one Celery Beat replica, and what's the risk of that?
5. What's the difference between RTO and RPO?
6. Why do read replicas exist if the primary can already handle reads?

*(Answers are all in the sections above — if you can answer these in your own words, you understand the architecture, not just memorized it.)*
