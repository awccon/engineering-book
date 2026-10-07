# Scalability, Reliability and Observability

Book IX gave Beacon the Azure mechanics: autoscaling rules, zone redundancy, Application Insights,
SLOs with burn-rate alerts, load tests and cost controls. This chapter steps back to the **thinking**
behind them, the parts that carry over to any platform:

- **Capacity**: how to estimate what a system needs before it falls over, and why systems degrade
  suddenly rather than gradually.
- **Failure modes**: the recurring ways real systems fail, most of which are self-inflicted.
- **Reliability as design**: limiting blast radius, degrading gracefully, changing safely.
- **Observability as a property of the design**, not a product you install.

---

## 1. The problem: systems fail at the edges of what you tested

Most systems work fine at the load and in the conditions their developers tried. They fail when
something is different: ten times the usual traffic after a marketing email, a dependency that's slow
instead of down, a deployment during a database failover, a customer who imports 400,000 tickets at once.

Three questions every experienced engineer asks about a system:

1. **How much can it handle, and what breaks first?** (capacity)
2. **When something fails, how much fails with it?** (reliability)
3. **When something is wrong, how quickly can we tell what and why?** (observability)

None of them can be answered by the code alone. They need numbers, experiments and instrumentation.

---

## 2. The mental model: queues everywhere

Every component that does work (a CPU, a thread pool, a connection pool, a database, a message
consumer) is a **queue**: work arrives, waits, gets served. Two laws explain most capacity behavior.

### Little's Law

For any stable system:

```text
L = λ × W

L = average number of items in the system (concurrency)
λ = arrival rate (throughput)
W = average time each item spends in the system (latency)
```

It's simple and universally true, and it answers practical questions:

- Beacon's API handles 400 requests/s with a mean latency of 50 ms, so on average **20 requests are in
  flight**. If a dependency slows down and latency rises to 2 s, in-flight requests rise to **800** at the
  same throughput. Do you have 800 threads, connections and memory for that? That's how a slow
  dependency becomes an outage (Chapter 3).
- The PostgreSQL connection pool has 50 connections and queries average 10 ms: the pool can sustain about
  **5,000 queries/s**. If queries average 100 ms, about 500/s.
- A consumer processes messages in 200 ms with 8 concurrent handlers: about **40 messages/s** per
  replica. A backlog of 100,000 takes 42 minutes on one replica.

### Utilization and the hockey stick

As a resource gets busier, waiting time doesn't rise linearly. In the simplest queueing model, waiting
time is proportional to `ρ / (1 − ρ)`, where ρ is utilization:

```text
utilization   50%   70%   80%   90%   95%   99%
wait factor    1×   2.3×   4×    9×   19×   99×

latency
   │                                         ╱
   │                                       ╱
   │                                    ╱
   │                               _ ─ ╯
   │ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─
   └──────────────────────────────────────── utilization
   0%                          70–80%      100%
```

> **🧱 Durable:** Systems don't degrade gracefully as they approach capacity; latency **explodes** near
> saturation. Plan to run critical resources at well under 100% (typically 50–70% at peak), and scale
> on leading signals (concurrency, queue length) rather than waiting for latency to rise.

This is also why a 10% traffic increase can cause a 10× latency increase on a resource that was already
at 90%.

### Amdahl and the serial bottleneck

Adding instances only helps the part of the work that parallelizes. If every request takes a row lock on
the same "organization counter", or goes through one single-threaded component, that part sets the
ceiling no matter how many replicas you add. The **Universal Scalability Law** adds a second effect:
coordination between instances (locks, cache invalidation, consensus) can make throughput go **down**
as you add more. Scaling out a component that contends on a shared resource can make things worse.

---

## 3. Capacity thinking: estimate, then measure

### Back-of-the-envelope estimates

Before building or load testing, rough numbers catch most bad designs. Useful reference magnitudes
(orders of magnitude, not benchmarks):

| Operation | Rough time |
|---|---|
| Main memory reference | ~100 ns |
| Read 1 MB sequentially from memory | ~10–50 µs |
| SSD random read | ~20–100 µs |
| Round trip within an Azure region (same zone) | ~0.2–1 ms |
| Simple indexed PostgreSQL query over the network | ~1–3 ms |
| Round trip between distant regions | ~70–150 ms |
| LLM call (short output) | ~0.5–5 s |

And the arithmetic of daily volumes: **1 million requests/day ≈ 12/s average**. Peaks are typically
3–10× the average for business applications (working hours, Monday mornings).

A Beacon estimate for a large customer:

```text
5,000 agents, 200,000 customers
Tickets: 50,000/day → 0.6/s average, ~5/s peak
Ticket views + list refreshes: 5,000 agents × 300/day = 1.5 M/day → 17/s average, ~120/s peak
Comments: 200,000/day → ~20/s peak
AI summaries: 1 per ticket view with cache hit rate 80% → ~25 model calls/s peak
                                                         ↑ this is the expensive, slow part
Storage: 50,000 tickets × ~20 KB (with comments) = 1 GB/day ≈ 365 GB/year + attachments in Blob Storage
Embeddings: 50,000 × 1,536 floats × 4 bytes ≈ 300 MB/day of vectors → index memory planning matters
```

Notice what the estimate reveals before any test: the relational load is modest (PostgreSQL handles this
on one well-sized server), the **AI calls dominate cost and latency**, and **vector index size** grows
fast enough to plan for. That's where design attention should go.

### Load testing

Estimates tell you where to look; load tests tell you the truth. Book IX, Chapter 7 covered the tooling
(Azure Load Testing, k6). The method that matters:

1. **Model real traffic**: the mix of endpoints, read/write ratio, think time, data volumes (a test
   against an empty database proves nothing about queries).
2. **Ramp up until something breaks**, and identify **what** broke first: CPU, connection pool, database
   locks, a dependency's rate limit, memory.
3. **Fix or accept that bottleneck**, then repeat. There's always a next one.
4. **Test failure under load**: kill a replica, fail over the database, make a dependency slow, during the
   test.
5. **Soak test**: hours at normal load find leaks and slow degradation that short tests miss.

### The scaling ladder

When a component reaches its limit, try the cheapest steps first:

```text
1. Fix the inefficiency     indexes, N+1 queries, payload sizes, allocations (Books I, IV)
2. Cache                    HybridCache, output cache, CDN (Book III, Chapter 7)
3. Scale up                 bigger instance: boring, effective, often cheapest in engineering time
4. Scale out stateless      more API replicas (Beacon is stateless; sessions live in the BFF cookie)
5. Move work async          queues and background processing (Chapter 4)
6. Read replicas            offload reporting and heavy reads (mind read-your-writes, Chapter 3)
7. Partition                split tables by time or tenant (PostgreSQL declarative partitioning)
8. Shard                    split data across databases by tenant (Citus, or application-level)
```

Each step up the ladder adds operational complexity. Many systems never need steps 7 and 8; plenty of
large businesses run on one well-tuned PostgreSQL primary with replicas.

> **🔄 Current (as of October 2026):** For PostgreSQL scale-out on Azure, **Azure Cosmos DB for PostgreSQL**
> (Citus) distributes tables across nodes by a distribution key, and **Azure Database for PostgreSQL
> Flexible Server** offers larger compute tiers, read replicas and elastic clusters. The details change
> often; check current limits before choosing, and measure whether you need distribution at all.

---

## 4. Failure modes: how systems actually fail

Outage reports from large companies are remarkably consistent. The same patterns recur:

### Changes cause most outages

Deployments, configuration changes, feature flag flips, certificate rotations, schema migrations and
infrastructure updates cause the majority of incidents in well-run systems. The best reliability
investment is usually **safer change** (section 5), not more redundancy.

### Cascading failure

One component slows down; callers hold resources longer (Little's Law); callers become slow; their
callers do the same. Retries add load. Within minutes everything is down, even though only one thing was
broken. Defenses: timeouts, bounded concurrency (bulkheads), circuit breakers, load shedding, retry
budgets (Chapter 3).

### Resource exhaustion

Something runs out: connections, file descriptors, threads, memory, disk, ephemeral ports, a quota, an
API rate limit, a certificate's validity. Usually it was slowly running out for weeks. Defenses: alert
on **saturation trends** (disk at 80% and growing), set limits explicitly so exhaustion happens in a
controlled place, and track quotas as part of capacity planning.

### Thundering herd

Many clients do the same thing at the same moment: a cache entry expires and 2,000 requests recompute it;
every client reconnects after a network blip; every instance's scheduled job fires at midnight UTC.
Defenses: request coalescing (HybridCache does this for cache misses), jittered expirations and
schedules, staggered reconnects with backoff.

### Metastable failure

A system that is stable under normal conditions is pushed into an overloaded state by a trigger (a short
outage, a cold cache), and **stays** overloaded after the trigger is gone, because the overload itself
(retries, timeouts doing wasted work, cache misses) sustains it. Recovery often requires shedding load
manually: blocking traffic, disabling retries, warming caches. Designing to avoid it means keeping
retry amplification bounded and making sure the system does less work, not more, when overloaded.

### Gray failure

A component is partly broken: one zone has 5% packet loss, one replica has a bad disk, one node's clock is
wrong. Health checks pass (they test the easy path), so traffic keeps going to it. Defenses: health
checks that test real dependencies (but carefully: Book III, Chapter 8 warned against liveness checks
that cascade), outlier detection in load balancers, and per-instance metrics so one bad instance stands
out.

### Data and correctness failures

The scariest outages don't produce errors: a bug that writes wrong data, a migration that corrupts a
column, a job that deletes too much. Availability monitoring doesn't catch them. Defenses: backups you
have actually restored (Book IV, Chapter 8), point-in-time recovery, soft deletes for critical data,
data validation jobs, and limits on destructive operations (a cleanup job that refuses to delete more
than 1% of rows without confirmation).

---

## 5. Reliability as a design property

### Redundancy, and its limits

Redundancy (multiple replicas, zones, regions) protects against **independent** failures: a host dies, a
zone loses power. It doesn't protect against **correlated** ones: a bad deployment goes to every replica,
a poison message crashes every consumer, a configuration error applies everywhere. Most serious outages
are correlated.

### Limiting blast radius

Since you can't prevent every failure, limit **how much** fails:

- **Bulkheads** per dependency and per tenant (Chapter 3).
- **Cells**: run several independent copies of the whole stack, each serving a subset of tenants. A
  bad deployment or a noisy tenant affects one cell. Large SaaS systems use cell-based architectures;
  Beacon might move its largest customers into dedicated cells when they need it.
- **Progressive delivery**: deploy to a canary (Book IX, Chapter 9), then one cell or region at a time,
  with automatic rollback on SLO regression.
- **Feature flags** to turn off a misbehaving feature without a deployment (Beacon's AI features are
  already behind flags, Book XI).
- **Rate limits per tenant**, so one customer's bulk import can't starve everyone else.

### Safe change

Since change causes most outages:

- **Small, frequent deployments** are safer than large, rare ones: each carries less risk and is easier to
  diagnose and roll back.
- **Every change reversible**: backward-compatible database migrations (expand/contract), versioned
  contracts, flags for behavior changes, configuration in source control.
- **Automated rollback** on health and SLO signals.
- **Configuration changes go through the same pipeline** as code: reviewed, tested, progressively rolled
  out. Many famous outages were configuration pushed globally at once.

### Graceful degradation

Decide in advance what the system does without each dependency (Chapter 3's dependency failure table),
and test it. Beacon's degraded modes: read-only mode during database failover, hidden AI features when
the model is unavailable, PostgreSQL full-text search as fallback for vector search, notification
backlog during email provider outages.

### Testing failure: chaos engineering and game days

You only know your system survives a failure if you've seen it survive. **Chaos engineering** injects
failures deliberately (kill instances, add latency, drop connections, fail over databases) in a
controlled way, first in staging, eventually in production with small blast radius. **Game days** are
scheduled exercises where the team practices responding to a simulated incident. Azure Chaos Studio and
tools like Toxiproxy make the injection easy; the valuable part is the **hypothesis** ("if the Redis cache
fails, p95 latency will rise below 500 ms and error rate won't change") and fixing what proves it wrong.

### SLOs as an architecture tool

Book IX defined SLIs, SLOs and error budgets. Their architectural role:

- **They set the target that justifies cost.** 99.9% (43 minutes of downtime a month) and 99.99% (4
  minutes) are very different architectures. The second typically needs multi-region active-active,
  automated failover and extremely careful change management. Most internal and B2B systems don't need
  it.
- **Availability composes multiplicatively for serial dependencies.** If Beacon's ticket creation depends
  synchronously on five components at 99.95% each, the best it can do is about 99.75%. To meet a higher
  target, remove synchronous dependencies (Chapter 4) or make them redundant.
- **Error budgets decide priorities.** Budget left: ship features faster. Budget exhausted: freeze risky
  changes and invest in reliability. This turns reliability arguments into data.

---

## 6. Observability as a design property

**Monitoring** answers questions you predicted ("is the error rate above 1%?"). **Observability** is the
ability to answer questions you **didn't** predict, about states you've never seen, from the outside,
without shipping new code. The difference matters because novel failures are exactly the ones that hurt.

### Signals

Book IX, Chapter 6 covered the three classic signals: **logs, metrics, traces**, collected with
OpenTelemetry. Two design ideas make them far more useful:

**Structured, high-context events.** Instead of many log lines per request ("starting", "loaded ticket",
"calling AI"), emit fewer, **wide** events with many fields: tenant, user role, ticket priority, feature
flags active, cache hit, model used, token counts, database time, outcome. With trace IDs on every one,
you can slice by any dimension: "Are slow requests concentrated in one tenant? One feature flag? One
replica? One model?"

```csharp
// Enrich the current trace span with business context; it lands on every telemetry item for the request
public static class TicketTelemetry
{
    public static void Annotate(Ticket ticket, ICurrentUser user)
    {
        var span = Activity.Current;
        if (span is null) return;
        span.SetTag("beacon.tenant", user.OrganizationId);
        span.SetTag("beacon.ticket.priority", ticket.Priority.ToString());
        span.SetTag("beacon.ticket.team", ticket.TeamId?.ToString());
        span.SetTag("beacon.user.role", user.Role);
    }
}
```

**Cardinality on purpose.** Metrics with high-cardinality labels (user ID, ticket ID) are expensive or
impossible in most metric systems; traces and structured logs handle them. Keep metrics for low-cardinality
aggregates (rate, errors, latency by endpoint and tenant tier), and use traces and events for the
details.

### Instrument at boundaries

The most valuable instrumentation is at the edges of each component: every incoming request, outgoing
call, message consumed and published, and job run. OpenTelemetry's automatic instrumentation for
ASP.NET Core, `HttpClient`, Npgsql, Service Bus and `Microsoft.Extensions.AI` (Book XI) covers most of
that. Add custom spans for business-significant work (the SLA scan, an AI agent's tool loop) and
counters for business events (tickets created, SLA breaches, AI suggestions accepted).

### Observability for asynchronous flows

Chapter 4's trace propagation through messages turns "the email was sent eight minutes later by a
different process" into one connected trace. Add **lag** metrics (outbox age, subscription backlog,
consumer delay) and **end-to-end** latency for business flows ("time from ticket resolved to customer
notified"), which no single component's metrics show.

### Debuggability is designed in

When designing a feature, ask: "When this goes wrong at 3 a.m., what will the on-call engineer need to
see?" Then make sure it's emitted. Correlation IDs in error responses (Problem Details with `traceId`,
Book III), audit logs for state changes, and the ability to turn up log verbosity for one tenant
without a deployment are all design decisions.

---

## 7. Incidents and learning

Reliability improves through how a team responds to and learns from incidents.

**During an incident:**

- **Mitigate first, diagnose later.** Roll back, fail over, disable the flag, shed load. Root cause can
  wait until users are unaffected.
- **Clear roles**: an incident commander who coordinates (and doesn't debug), people investigating, and
  someone communicating with stakeholders and customers.
- **A timeline as you go**, in the incident channel: what was observed, what was tried, when.

**After an incident, a blameless postmortem:**

- **What happened**, with a timeline and impact (users affected, duration, error budget consumed).
- **Contributing factors**, plural. Complex system failures rarely have one root cause; "the engineer ran
  the wrong command" is never the end of the analysis. Ask why the command was possible, why it looked
  right, and why nothing caught it.
- **What went well** (detection, mitigation, tools).
- **Action items** with owners and dates, focused on making the failure **less likely, less severe or
  faster to detect**. Track them to completion.

> **🧱 Durable:** **Blamelessness is not about being nice; it's about getting accurate information.** If
> people fear blame, they hide details, and the organization learns the wrong lessons. Assume everyone
> acted reasonably given what they knew, and ask what made the failure possible.

---

## 8. In practice: a capacity and resilience review of Beacon

A large customer (5,000 agents, the estimate in section 3) is about to onboard. The team runs a review.

**Step 1: Estimate.** Section 3's numbers: peak ~150 requests/s on the API, ~25 AI calls/s, ~1 GB/day
of ticket data, ~300 MB/day of embeddings.

**Step 2: Load test** with production-like data (2 years of synthetic tickets for 5,000 agents) and the
real traffic mix. Results, first bottleneck to last:

| Load | Symptom | Bottleneck | Fix |
|---|---|---|---|
| 60 req/s | p95 of ticket list jumps to 2 s | Missing index for the new "my team's open tickets by SLA due" filter | Composite partial index (Book IV, Chapter 4) |
| 110 req/s | Errors: "connection pool exhausted" | 4 replicas × 100-connection pools > server's max connections; connection storms on scale-out | PgBouncer-style pooling (Flexible Server's built-in PgBouncer), pool size 30 per replica |
| 140 req/s | AI summary p95 9 s, 429s from the model | Model deployment's tokens-per-minute quota | Cache summaries by ticket version, summarize on change (async) not on view, request quota increase, cheaper model for short tickets |
| 200 req/s | SignalR fan-out CPU spikes | Broadcasting every update to the whole team group | Azure SignalR Service, finer groups (per queue view) |
| soak 6 h | Memory grows 40 MB/h on worker | Unbounded in-memory dictionary caching tenant settings in the worker | `MemoryCache` with a size limit and expiry |

**Step 3: Failure tests under load.**

- Kill one API replica: brief error spike from in-flight requests; added connection draining and checked
  `terminationGracePeriodSeconds` (Book VIII).
- PostgreSQL failover: 35 s of write errors. The UI showed a generic error; added a read-only banner
  driven by health status, and idempotency keys (Chapter 3) let clients retry safely.
- AI model latency +10 s: circuit breaker opened after 30 s, summaries hidden. Good.
- Redis down: latency +40%, no errors. Good.

**Step 4: Blast radius.** The new customer's bulk import of 400,000 historical tickets would flood the
outbox and every downstream consumer. Fixes: imports go through a separate, rate-limited import queue;
imported tickets are marked as historical and skip notifications, webhooks and AI triage; indexing for
imported data runs at low priority.

**Step 5: Observability gaps.** During the tests, the team couldn't easily tell which tenant caused load.
Added `beacon.tenant` to spans and per-tenant-tier request metrics, plus an end-to-end "resolved →
notified" latency metric.

**Step 6: Record the result**: capacity numbers, headroom (target: peak at ≤60% of tested limit),
known limits, and the next bottleneck (vector index memory in about 9 months at current growth), in an
ADR (Chapter 2) with a date to revisit.

> **🔍 Investigation:** Why did the connection pool fail at 110 req/s and not earlier? Little's Law: at
> 110 req/s with ~8 database calls of ~4 ms each per request, the API needs only about 4 connections
> busy on average. The real cause was **scale-out**: autoscaling added replicas, each opening its pool
> eagerly, briefly demanding more connections than PostgreSQL allowed, while slow queries from the
> missing index held connections longer. Two interacting causes, neither sufficient alone, which is
> typical.

---

## 9. What can go wrong

- **No capacity numbers**: "it's fast on my machine" as the scaling plan.
- **Load tests against empty databases** or unrealistic traffic mixes.
- **Running hot**: resources at 85–90% at normal peak, leaving no headroom for spikes or failover.
- **Scaling the wrong tier**: adding API replicas when the database is the bottleneck, which makes it
  worse (more connections, more contention).
- **Redundancy as the only reliability strategy**, ignoring correlated failures from changes.
- **Health checks that lie**: always green, or so strict they take everything down during a dependency
  blip.
- **Alerting on everything** (alert fatigue) or on causes rather than symptoms.
- **Dashboards nobody looks at** and telemetry nobody can query in an incident.
- **Postmortems without follow-through**: the same incident, three times a year.
- **Ignoring data correctness**: perfect availability while silently writing wrong data.

---

## 10. When not to use it

> **🧭 When not to use it:** Match investment to need. An internal tool used by 30 people doesn't need
> cells, chaos experiments, multi-region failover or a 99.99% SLO. Use managed services' defaults, basic
> health checks and a couple of alerts, and spend the time on features. Likewise, don't build sharding,
> partitioning or complex caching for load you don't have; estimate first, and climb the scaling ladder
> only when measurements say so.

---

## 11. How an experienced engineer thinks about this

- **Does the arithmetic first.** Little's Law and a back-of-the-envelope estimate catch most bad designs
  before any code exists.
- **Knows systems fail suddenly near saturation**, and keeps headroom on purpose.
- **Finds the actual bottleneck by measurement**, fixes it, then looks for the next one.
- **Climbs the scaling ladder from the bottom**: efficiency, caching and bigger machines before
  distribution.
- **Treats change as the main risk** and invests in small, reversible, progressive deployments.
- **Designs for blast radius**, not just redundancy.
- **Tests failure deliberately** rather than hoping redundancy works.
- **Builds observability in** by asking what the on-call engineer will need to see.
- **Runs blameless postmortems** and makes sure action items get done.

---

## 12. Check yourself

**Questions**

1. State Little's Law and use it to explain how a slow dependency can exhaust a service's resources.
2. Why does latency rise sharply as utilization approaches 100%? What headroom would you target?
3. What does the Universal Scalability Law add to Amdahl's Law?
4. List the steps of the scaling ladder in order. Why start at the bottom?
5. Describe cascading failure, thundering herd, metastable failure and gray failure, with a defense
   for each.
6. Why does redundancy not protect against most serious outages?
7. How do SLOs influence architecture? Compute the best-case availability of five serial dependencies
   at 99.95%.
8. What's the difference between monitoring and observability?
9. What makes a postmortem blameless, and why does it matter?

**Exercises**

1. Do a back-of-the-envelope capacity estimate for a system you work on: requests/s at peak, database
   queries/s, storage growth per year, and the most expensive operation.
2. Run a load test of Beacon's ticket list and detail endpoints with realistic data and find the first
   bottleneck. Record the result.
3. Use Toxiproxy (or Azure Chaos Studio) to add 2 s of latency to PostgreSQL during a load test.
   Observe what happens to in-flight requests, thread pool and connection pool, and explain it with
   Little's Law.
4. Add tenant and feature-flag context to Beacon's spans, then answer: "Which tenant generated the most
   AI calls yesterday, and what was their p95 latency?"
5. Write a blameless postmortem for an incident you've experienced (or the Beacon investigation in
   section 8).

**Interview-style questions**

- "How would you estimate the capacity needed for a system with 10 million daily active users?"
- "Your service's latency doubled but traffic only went up 10%. What might be happening?"
- "How do you prevent cascading failures in a microservices system?"
- "Tell me about an incident you were involved in. What did you learn?"
- "What's the difference between monitoring and observability?"

---

## 13. Going deeper

- Betsy Beyer et al., *Site Reliability Engineering* and *The Site Reliability Workbook* (Google; free
  online at [sre.google](https://sre.google/books/)) — SLOs, cascading failures, postmortems.
- Michael Nygard, *Release It!* (2nd ed.) — stability patterns and antipatterns from real outages.
- Charity Majors, Liz Fong-Jones and George Miranda, *Observability Engineering* — wide events and
  high-cardinality debugging.
- Neil Gunther's Universal Scalability Law and Marc Brooker's blog posts on queueing and metastable
  failures.
- Bronson et al., "Metastable Failures in Distributed Systems" (HotOS 2021).
- [The Amazon Builders' Library](https://aws.amazon.com/builders-library/) — shuffle sharding,
  cell-based architecture, load shedding.
- The [Learning from Incidents](https://www.learningfromincidents.io) community.

**Next:** [Security Engineering](06-security-engineering.md) looks at the system from an attacker's point
of view: threat modeling, defense in depth and the software supply chain.
