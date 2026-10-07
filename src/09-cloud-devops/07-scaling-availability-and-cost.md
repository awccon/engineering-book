# Scaling, Availability and Cost

Three questions every production system eventually faces: *Can it handle more load?* *Does
it stay up when things fail?* *Can we afford it?* They pull against each other. More
replicas, more zones, more regions and bigger databases improve scalability and
availability, and cost more. Cheaper configurations save money and give up headroom or
resilience.

This chapter covers how systems scale, how to design for failure, how much availability is
worth paying for, and how to keep cloud costs under control. It ends with Beacon's
deliberate trade-offs, written down.

---

## 1. The problem: growth, failure and the bill

- Beacon's biggest customer doubles its support team; Monday mornings now have three times
  the traffic.
- A zone in West Europe has a power incident; half the replicas disappear.
- Finance asks why the cloud bill grew 40% last quarter.

Each needs a design answer, not a heroic response at the time.

---

## 2. The mental model: scaling dimensions

### Vertical vs horizontal

| | Vertical (scale up) | Horizontal (scale out) |
|---|---|---|
| How | Bigger machine (more CPU, memory) | More machines/replicas |
| Limit | The largest instance size | Mostly the architecture |
| Complexity | Low | Requires statelessness, load balancing, coordination |
| Downtime to change | Often a restart | None (add/remove replicas) |
| Typical for | Databases, legacy apps | Stateless web/API tiers, workers |

Beacon's API, BFF and workers scale **horizontally** (they're stateless: Book VIII, Chapter 4).
PostgreSQL scales **vertically**, plus read replicas for reads (Book IV, Chapter 8). That's
the common shape of most web applications: a horizontally scaled stateless tier in front of
a vertically scaled stateful core.

### Where bottlenecks move

Scaling one tier moves the bottleneck to the next:

```text
 more API replicas ─► more concurrent DB queries ─► DB CPU / connections saturate
 ─► add PgBouncer, optimize queries, scale DB up, add read replica, cache ─► ...
```

Before scaling anything, find the actual bottleneck (Book III, Chapter 10; Book IV,
Chapter 6). Scaling the wrong tier increases cost without improving anything.

> **🧱 Durable:** A system's capacity is set by its most constrained shared resource,
> usually the database. Horizontal scaling of stateless tiers is easy; the hard problems are
> always in shared state.

---

## 3. Autoscaling

Autoscaling adjusts capacity to load automatically.

| Trigger | Good for | Caveat |
|---|---|---|
| **HTTP concurrency / request rate** | Web and API tiers | Reacts to load directly; tune the target per replica from load tests |
| **CPU / memory** | CPU-bound workloads | Lagging indicator for I/O-bound apps; memory rarely drops after scaling |
| **Queue length / lag** (KEDA) | Workers | Scale workers to the backlog; scale to zero when empty |
| **Schedule** | Predictable peaks (Monday 08:00) | Pre-scale before the load arrives |

Principles:

- **Load test to find per-replica capacity** (e.g. one API replica handles 150 requests/second
  at p95 < 300 ms), then set targets with headroom.
- **Scale out fast, scale in slowly**, to avoid flapping.
- **Set maximums** to protect downstream dependencies (20 API replicas × 30 connections = 600
  database connections: more than PostgreSQL can take without PgBouncer).
- **Minimum replicas ≥ 2** in production for availability; scale to zero in non-production
  to save money.
- **Warm-up**: new replicas start cold (JIT, caches; Book I, Chapter 1); readiness probes
  should only pass when they can serve well.

### Load testing

Use **Azure Load Testing** (managed JMeter/Locust), k6 or similar to:

- find per-replica capacity and the breaking point,
- verify autoscaling reacts in time,
- reveal the bottleneck (it's rarely where you expect),
- check behavior under overload (graceful degradation vs collapse).

Run them against a production-like environment, with realistic data volumes and request mix.

---

## 4. Availability: designing for failure

### How much availability?

| Availability | Downtime per month | Typical requirement |
|---|---|---|
| 99% | ~7.2 hours | Internal tools |
| 99.9% ("three nines") | ~43 minutes | Most business SaaS |
| 99.95% | ~22 minutes | Important customer-facing services |
| 99.99% ("four nines") | ~4.3 minutes | Payments, critical infrastructure |
| 99.999% | ~26 seconds | Telecom-grade; very expensive |

Each additional nine typically requires a step change in architecture and cost. The right
target comes from the business: what does an hour of downtime cost, and what are customers
promised (contractual SLAs)?

### Composite availability

A request depending on several components in series is only as available as their product:

```text
 Front Door 99.99% × BFF 99.95% × API 99.95% × PostgreSQL (zone-redundant) 99.99% × Redis 99.9%
 ≈ 99.78%    (≈ 1.6 hours of downtime per month, if failures are independent)
```

Ways to raise it: redundancy (more replicas, zones), removing hard dependencies (if Redis is
down, fall back to the database instead of failing; Book III, Chapter 7), and graceful
degradation.

### Resilience patterns

| Pattern | Protects against | Example in Beacon |
|---|---|---|
| **Redundancy** across zones | Instance and zone failures | ≥ 2 replicas per app, zone-redundant DB |
| **Timeouts** | Hanging dependencies | HttpClient, DB command, proxy timeouts |
| **Retries with back-off and jitter** | Transient failures | Standard resilience handler (Book III, Chapter 1); EF Core retries |
| **Circuit breaker** | Hammering a failing dependency | Stop calling the email provider for 30 s after repeated failures |
| **Bulkheads** | One failure exhausting shared resources | Separate worker app; separate connection pools |
| **Graceful degradation** | Non-critical dependency outages | Live updates off → page still works; search down → show recent tickets |
| **Queue-based load leveling** | Spikes | Exports and notifications via queues (Book III, Chapter 8) |
| **Idempotency** | Retries causing duplicates | Idempotency keys, idempotent consumers |
| **Health-based routing** | Unhealthy instances | Readiness probes; Front Door health probes |

### Multi-region

Zone redundancy handles data center failures. **Region** failures are rare but real (and so
are region-wide service incidents). Options, in increasing cost and complexity:

| Strategy | RTO | RPO | Cost | Complexity |
|---|---|---|---|---|
| **Backup and restore** in another region | Hours | Minutes–hours (geo-backup lag) | Low | Low |
| **Pilot light / warm standby** (infrastructure defined, DB replica, apps scaled down) | Tens of minutes | Seconds–minutes | Medium | Medium |
| **Active-passive** (full standby, automatic failover) | Minutes | Seconds | High | High |
| **Active-active** (both regions serve traffic) | ~0 | ~0 to seconds | Very high | Very high (data conflicts, consistency) |

> **🧭 When not to go multi-region:** Active-active across regions is among the hardest
> things in distributed systems, mostly because of data: writes in two regions must be
> replicated and reconciled (Book XIII, Chapter 3). Unless the business genuinely requires
> near-zero regional RTO, a zone-redundant single region plus a tested regional recovery plan
> (infrastructure as code + geo-backups) is far cheaper and simpler.

### Testing failure

Resilience you haven't tested is a hope. **Chaos engineering** (Azure Chaos Studio, or
simple scripts) deliberately injects failures in staging and, carefully, in production: kill
replicas, add latency to the database, block a dependency, fail over the database. Each
experiment verifies an assumption ("the API keeps serving when Redis is unavailable").

---

## 5. Cost: the mental model

Cloud cost = **what you provision** × **how long** + **what you consume**:

| Cost driver | Examples | Levers |
|---|---|---|
| Compute | vCPU/memory-seconds of replicas, VM hours, plan instances | Right-size, autoscale, scale to zero, reservations/savings plans, Spot for interruptible work |
| Databases | vCores, storage, IOPS, HA standby (doubles compute), replicas, backup storage | Right-size, Burstable tiers for non-prod, reserved capacity, stop dev servers at night |
| Storage | GB-months by tier, transactions | Lifecycle policies (Chapter 4), cool/archive tiers |
| Networking | **Egress** (data leaving Azure), cross-zone and cross-region traffic, Front Door requests | CDN caching, compression, keeping chatty components together |
| Observability | Log/trace ingestion and retention | Sampling, log levels, retention tiers (Chapter 6) |
| Managed services | Front Door, Key Vault operations, Redis, SignalR units | Choose tiers deliberately; review usage |

### FinOps practices

**FinOps** is the practice of making cloud spending visible, accountable and optimized:

1. **Visibility**: tags (`app`, `env`, `owner`, `costCenter`; Chapter 1) and Azure Cost
   Management views per application and environment.
2. **Budgets and alerts**: a budget per subscription/resource group, with alerts at 50%, 80%,
   100% and forecasted overruns.
3. **Accountability**: each team sees and owns its costs.
4. **Optimization loop**: monthly review of top cost drivers; Azure Advisor recommendations
   (idle resources, oversized SKUs, reservation opportunities).
5. **Unit economics**: cost per customer, per ticket, per active user. "The bill grew 40%" is
   alarming; "cost per ticket fell 10% while volume grew 55%" is good news.

### Common waste

- Non-production environments running 24/7 at production size.
- Orphaned resources: disks, IPs, old environments, forgotten test databases.
- Over-provisioned databases "just in case."
- Logging everything at `Information` with long retention.
- Uncached static assets and chatty cross-region calls generating egress.

> **⚠️ What can go wrong:** Cost optimization that silently removes resilience: dropping to
> one replica, disabling zone redundancy or HA, shortening backup retention to save money.
> Make availability and cost trade-offs **explicitly**, with the people who own the
> business risk.

---

## 6. In practice: Beacon's capacity, availability and cost decisions

**Targets** (agreed with the product owner):

- Availability SLO 99.9% for the agent and customer apps (matches the SLA offered to customers).
- RPO 5 minutes, RTO 1 hour for a zone failure: automatic; for a region failure: RTO 8 hours
  via documented recovery (Book IV, Chapter 8).

**Capacity** (from load tests in staging with 2 million tickets):

- One API replica (1 vCPU, 2 GB): ~180 requests/s at p95 < 250 ms for the typical mix.
- Peak production load (Monday 08:00–10:00): ~600 requests/s → 4 replicas + headroom.
- Autoscale: HTTP concurrency target 50 per replica; min 2, max 12 (12 × 30 pool connections,
  through PgBouncer, within database limits).
- Scheduled pre-scaling to 4 replicas at 07:45 on weekdays.
- Bottleneck found: database CPU on the ticket list query at ~900 requests/s; fixed with the
  partial index from Book IV, Chapter 4 and a 30-second cache on queue counts.

**Availability design**:

- Zone-redundant Container Apps environment; ≥ 2 replicas of BFF and API.
- Zone-redundant PostgreSQL HA; zone-redundant Redis; ZRS storage.
- Degradation: Redis down → HybridCache falls back to L1 and database; SignalR down → UI shows
  "updates paused" (Book VI, Chapter 3) and polls every 60 s; email provider down → outbox
  retries, notifications delayed, nothing lost.
- Quarterly chaos experiment in staging: kill all replicas in one zone; fail over PostgreSQL;
  block Redis.
- Regional recovery runbook: deploy infrastructure (Bicep, Chapter 9) to North Europe, restore
  geo-redundant backup, switch Front Door origin. Rehearsed twice a year.

**Cost** (illustrative monthly breakdown for production, to be checked against the Azure
pricing calculator for current prices):

| Component | Share |
|---|---|
| PostgreSQL (4 vCores, zone-redundant HA doubles compute) | ~40% |
| Container Apps (BFF, API, workers) | ~25% |
| Front Door Premium + WAF | ~12% |
| Log Analytics / Application Insights | ~10% |
| Redis, SignalR, storage, Key Vault, networking | ~13% |

Decisions:

- **Non-production**: scale to zero; Burstable PostgreSQL without HA, stopped nights and
  weekends; 7-day log retention; Front Door Standard shared across non-prod environments.
- **Reservations**: 1-year reserved capacity for production PostgreSQL after three months of
  stable usage.
- **Observability**: 25% trace sampling (always keeping errors and slow requests); debug logs
  off; 30-day interactive retention, then archive.
- **Budgets**: alerts at 80% and 100% of forecast per environment; monthly cost review in the
  team's operations meeting, tracking **cost per resolved ticket**.

These decisions live in `docs/operations.md` next to the database checklist (Book IV, Chapter
8), so they can be reviewed and revisited.

---

## 7. What can go wrong

- **Scaling the wrong tier** without finding the bottleneck.
- **Autoscaling without maximums**, overwhelming the database.
- **Single replicas in production.**
- **Untested failover and recovery.**
- **Availability targets nobody agreed on**, or promised SLAs the architecture can't meet.
- **Multi-region complexity** without a business need.
- **Silent cost growth**: no tags, no budgets, no review.
- **Cost cuts that remove resilience** without anyone deciding to accept the risk.

---

## 8. How an experienced engineer thinks about this

- **Measure capacity; don't guess it.** Load tests, per-replica numbers, known bottlenecks.
- **Stateless tiers scale out; state is the hard part.**
- **Availability is a business decision** with an engineering price for each nine.
- **Design for failure and test it.**
- **Cost is a design dimension**, tracked per unit of business value.

---

## 9. Check yourself

**Questions**

1. Compare vertical and horizontal scaling. Which fits stateless APIs, and which databases?
2. Why do autoscaling maximums matter?
3. How much monthly downtime does 99.9% allow? How do serial dependencies affect availability?
4. Name five resilience patterns and what each protects against.
5. Compare backup-and-restore, warm standby and active-active for regional failures.
6. What are the main cloud cost drivers, and one lever for each?
7. Why track unit economics rather than the total bill?

**Exercises**

1. Load-test Beacon.Api in staging and determine per-replica capacity and the first bottleneck.
2. Configure KEDA-based scaling for the outbox worker on backlog size, and scale-to-zero in
   non-production.
3. Run a chaos experiment: block Redis for Beacon and verify graceful degradation.
4. Set up Azure budgets and cost alerts for Beacon's resource groups, and build a cost view by
   tag.

**Interview-style questions**

- "How would you design an application to handle 10× traffic?"
- "How do you make a system highly available? What does it cost?"
- "How would you reduce a cloud bill without hurting reliability?"

---

## 10. Going deeper

- [Azure Well-Architected Framework: Reliability, Performance Efficiency, Cost Optimization](https://learn.microsoft.com/azure/well-architected/)
- [Azure Architecture Center: Cloud design patterns](https://learn.microsoft.com/azure/architecture/patterns/)
- Michael Nygard, *Release It!* — stability patterns and anti-patterns.
- [FinOps Foundation](https://www.finops.org/)

**Next:** [Chapter 8 — CI/CD](08-ci-cd.md) automates building, testing and deploying Beacon.
