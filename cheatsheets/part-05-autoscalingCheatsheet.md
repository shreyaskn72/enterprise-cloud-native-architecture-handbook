# Part 5 – Autoscaling Cheatsheet

## 1. The Golden Rule
> Scale only as much as **downstream systems** can safely handle.
Bottleneck moves downstream if one layer scales without the next: `Flask → RabbitMQ → Celery → MySQL`

## 2. Three Autoscalers — Who Does What

| Component | Trigger | Scales |
|---|---|---|
| **HPA** | CPU / Memory / Custom metrics | API pod replicas |
| **KEDA** | Queue length / Events | Worker replicas |
| **Cluster Autoscaler** | Pending pods | AKS nodes |

- Request path: `Traffic ↑ → HPA → more pods → nodes full? → Cluster Autoscaler`
- Background path: `Queue ↑ → KEDA → more workers → nodes full? → Cluster Autoscaler`

## 3. Key Formulas (memorize these)

**HPA:**
```
Desired Replicas = Current Replicas × (Current Metric / Target Metric)
```
Example: 10 pods, CPU 84% vs target 70% → 10 × (84/70) = **12 pods**

**KEDA:**
```
Workers Needed = Queue Length / Messages handled per Worker
```
Example: 25,000 msgs / 100 per worker = **250 workers**

**DB Connection Budget (the constraint that overrides everything):**
```
(API Pods × Pool Size) + (Worker Pods × Pool Size) + Reserved ≤ DB Max Connections
```
Example: (60×10) + (125×5) = 1,225 > 1,200 max → **architecture fails**

## 4. HPA Metrics — Quick Ranking

| Metric | Use? | Note |
|---|---|---|
| CPU | ✅ Primary | Baseline, misleading if bursty |
| Memory | ✅ Primary | Watch for leaks |
| Request Rate | ✅ Secondary | Good for API servers |
| Response Time | ⚠️ Caution | Needs careful tuning |
| Queue Length | ❌ Never | That's KEDA's job |

Recommended targets: **CPU 70%, Memory 75%**

## 5. KEDA — Why Queue Length

- Direct signal of backlog (unlike CPU)
- Auto scale-down when queue drains
- Rough scaling curve: <50 msgs → 2 workers | 5,000 → 50 | 50,000 → 200

## 6. Always Set Min/Max Boundaries

| Component | Min | Max |
|---|---|---|
| Flask | 5 | 60 |
| Workers | 2 | 125 |
| Nodes | 3 | 12 |

Never leave max replicas unlimited — max must derive from **DB connection budget**, not a guess.

## 7. Preventing Flapping (Oscillation)

| Setting | Value | Purpose |
|---|---|---|
| HPA Stabilization Window | 300s | No scale-down for 5 min after scale-up |
| KEDA Cooldown Period | 300s | Prevent rapid scale-down to floor |
| KEDA Polling Interval | 30s | How often queue length is checked |

## 8. Cost Logic
More pods = better performance but higher cost. Autoscaling **lowers total cost** vs. static over-provisioning because capacity only grows when needed.

## 9. Load Test Scenarios (5 to know)

1. **Baseline** – 500 RPS sustained → no unnecessary scaling
2. **Spike** – 500→5,000 RPS → HPA + Cluster Autoscaler react in 1–2 min, DB stays healthy
3. **Queue burst** – 100k messages → KEDA scales up, drains, scales back down
4. **Node failure** – kill a node mid-load → pods reschedule, RabbitMQ quorum survives, no outage
5. **Connection exhaustion** – lower `max_connections` in test → pool + circuit breaker handle gracefully, alerts fire

## 10. Common Mistakes → Fixes

| Mistake | Fix |
|---|---|
| Unlimited HPA replicas | Cap max by connection budget |
| CPU-only worker scaling | Use KEDA + queue metrics |
| No resource requests/limits | Always define them |
| One node pool for everything | Use dedicated node pools |
| No stabilization window | Configure cooldown/stabilization |
| Ignoring DB limits | Capacity-plan from DB outward |

## 11. One-Line Mental Model
```
Business Demand → API Traffic + Queue Growth → HPA + KEDA (pods)
→ Cluster Autoscaler (nodes) → Database Capacity (hard ceiling)
```

**Exam takeaway:** The bottleneck shifts over time — CPU-bound at low traffic, DB-bound at scale. Autoscaling requires continuous tuning, not "set and forget."
