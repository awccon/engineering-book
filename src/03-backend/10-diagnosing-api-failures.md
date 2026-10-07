# Diagnosing API Failures

Sooner or later, you'll get the message: *"The API is down,"* or *"Customers say tickets
won't save,"* or *"Everything is slow since this morning."* What separates experienced
engineers in that moment isn't knowing the answer immediately. It's having a **method**:
a calm, systematic way to go from a vague symptom to a confirmed cause, using evidence
rather than guesses, while limiting the damage.

This chapter is that method, applied to web APIs, with a catalog of common failure
patterns and how to recognize each.

---

## 1. The problem: production is not your laptop

Debugging locally is easy: you can reproduce, attach a debugger, step through code. In
production you usually can't:

- the failure depends on real data, real load or a real dependency,
- it may be intermittent,
- you can't pause the process serving thousands of users,
- the evidence is whatever the system recorded *before* you started looking.

So production debugging is mostly **investigation of evidence** (logs, metrics, traces,
dumps) plus **careful experiments**, and it starts long before the incident: with the
observability you built in.

---

## 2. The mental model: mitigate first, then diagnose

During an incident there are two goals, in this order:

1. **Restore service** (mitigate): roll back, fail over, scale out, disable a feature
   flag, block an abusive client. You don't need to understand the root cause to roll back
   the deployment that started it.
2. **Understand and fix** (diagnose): find the root cause, fix it properly, prevent
   recurrence.

> **🧱 Durable:** "What changed?" is the most productive first question in any incident.
> Most production failures are caused by a change: a deployment, a configuration update,
> a dependency release, a data migration, a traffic shift, a certificate expiring. Check
> the change log before theorizing.

### The diagnostic loop

```text
  Observe symptoms ─► Scope the blast radius ─► Form hypotheses ─► Test the cheapest one ─┐
        ▲                                                                                 │
        └──────────────────── refine with new evidence ◄──────────────────────────────────┘
```

**Scope** the problem with questions that split the search space:

| Question | Why it helps |
|---|---|
| All requests or some? Which endpoints? | Code path vs infrastructure |
| All users or some? One tenant, one region, one client version? | Data- or client-specific |
| All instances or one? | A bad node vs a bad build |
| Since when? Gradual or sudden? | Correlate with changes; leak vs deploy |
| Errors or latency? Which status codes? | 4xx = clients/input; 5xx = us; timeouts = dependencies or saturation |

---

## 3. The three signals

Observability rests on three kinds of telemetry (Book IX, Chapter 6 sets them up with
OpenTelemetry):

| Signal | Answers | Example |
|---|---|---|
| **Metrics** | *Is* something wrong, and how much? | Request rate, error rate, p95 latency, CPU, thread pool queue |
| **Logs** | *What* happened in a specific case? | "Failed to save ticket T-42: unique constraint violation" |
| **Traces** | *Where* did the time go across components? | Request → API 1.8s → DB query 1.7s |

A practical starting dashboard for any API is the **RED** method per endpoint:
**R**ate, **E**rrors, **D**uration (p50, p95, p99). For resources (CPU, memory,
connections), the **USE** method: **U**tilization, **S**aturation, **E**rrors.

> **⚠️ What can go wrong:** Averages hide problems. An average latency of 120 ms can mean
> 99% of requests take 50 ms and 1% take 7 seconds. Always look at percentiles (p95, p99)
> and at the error rate, not just the mean.

---

## 4. Reading the symptoms: status codes as clues

| Symptom | First suspects |
|---|---|
| Spike in **400/422** | A client release sending different payloads; a contract change on your side |
| Spike in **401** | Expired or rotated signing keys; wrong audience/issuer config; clock skew; identity provider outage |
| Spike in **403/404** | An authorization change; a routing change; a bad deployment of the frontend calling wrong URLs |
| **405 / 415** | Method or content-type mismatch after a client or proxy change |
| **429** | Your rate limits, or a client retry storm |
| **500** | Unhandled exceptions: read the logs for the exception type |
| **502 Bad Gateway** | The proxy got no valid response: app crashed, restarted, wrong port, or closed the connection |
| **503** | App or proxy reporting unavailability: overload, failing health checks, all instances removed |
| **504 Gateway Timeout** | App too slow for the proxy's timeout: slow dependency, thread pool starvation, lock contention |
| Requests hang, no response | Deadlock, starvation, connection pool exhaustion, a dependency without a timeout |

---

## 5. A catalog of common API failures

### Dependency slowness or failure

**Symptoms:** latency rises on endpoints that use a particular dependency; traces show
time spent in database or HTTP spans; errors mention timeouts.

**Investigate:** trace waterfall for slow requests; the dependency's own metrics; whether
retries are amplifying the load.

**Mitigate:** timeouts, circuit breakers, fallbacks (cached data), shedding load.

### Database connection pool exhaustion

**Symptoms:** `InvalidOperationException: Timeout expired. The timeout period elapsed prior
to obtaining a connection from the pool.` (or Npgsql's equivalent). Latency spikes, then
errors.

**Causes:** connections not disposed (missing `using`); long transactions; slow queries
holding connections; sync-over-async blocking threads that hold connections; too many
instances × pool size exceeding the database's connection limit.

**Investigate:** database active connections (`pg_stat_activity` in PostgreSQL); slow
query logs; code paths opening connections without disposal.

### Thread pool starvation

**Symptoms:** latency high, CPU low, the database looks fine. Thread count climbing
slowly; requests time out at the proxy (504). Covered in depth in Book I, Chapter 9.

**Investigate:** `dotnet-counters` (thread pool queue length, thread count);
`dotnet-stack` to find threads blocked in `.Result`, `.Wait()`, or synchronous I/O.

### Memory leak

**Symptoms:** memory grows steadily over hours or days; eventually OOM kills or
container restarts (exit code 137); latency spikes from frequent gen2 GCs before that.

**Investigate:** `dotnet-counters` GC heap size over time; `dotnet-gcdump` comparison
between two points in time; look for growing collections and their roots (Book I,
Chapter 11).

### Bad deployment

**Symptoms:** errors begin exactly at a deployment time; sometimes only on new instances
during a rolling deploy.

**Mitigate:** roll back first. **Investigate:** diff the release (code, configuration,
packages), check startup logs for configuration validation failures.

### Configuration and secrets

**Symptoms:** sudden auth failures, connection failures or feature changes with no code
deployment. Expired certificates, rotated passwords, a changed environment variable.

**Investigate:** configuration change history; certificate expiry dates; startup logs.
`ValidateOnStart` (Chapter 3) turns many of these into immediate, obvious startup failures.

### Retry storms

**Symptoms:** a brief dependency hiccup turns into a sustained outage. Request volume to
the dependency multiplies.

**Cause:** every layer retries (client × gateway × API × HTTP client), multiplying load on
a struggling dependency. **Fix:** retry at one layer, with exponential back-off and jitter,
budgets and circuit breakers.

### Data-specific failures

**Symptoms:** only certain records fail: one customer, one ticket. A `NullReferenceException`
on legacy data, a string longer than a new limit, an unexpected enum value, an encoding
issue.

**Investigate:** logs with the failing IDs; reproduce with a copy of the specific record
(sanitized) locally.

---

## 6. Tools for live investigation

### Logs and traces

Start with the trace ID. If a user reports an error, the problem-details response
(Chapter 2) includes `traceId`; search logs and traces for it. With distributed tracing,
one ID shows every span across services.

### The .NET diagnostic tools

These attach to a running process (locally, in a container via `docker exec`, or through
sidecars in Kubernetes):

```bash
dotnet-counters monitor -p <pid> --counters System.Runtime,Microsoft.AspNetCore.Hosting
dotnet-trace collect -p <pid> --duration 00:00:30          # CPU samples + events
dotnet-stack report -p <pid>                               # current managed stacks of all threads
dotnet-dump collect -p <pid>                               # full dump for offline analysis
dotnet-gcdump collect -p <pid>                             # heap snapshot
```

`dotnet-dump analyze` lets you inspect a dump offline: `clrstack -all`, `dumpheap -stat`,
`gcroot`, `pe` (print exception). A dump taken at the moment of the problem is often worth
more than hours of guessing.

### Health checks

```csharp
builder.Services.AddHealthChecks()
    .AddNpgSql(connectionString, name: "database", tags: ["ready"]);

app.MapHealthChecks("/health/live", new() { Predicate = _ => false });                   // process is up
app.MapHealthChecks("/health/ready", new() { Predicate = c => c.Tags.Contains("ready") }); // can serve traffic
```

- **Liveness**: "is the process alive?" Failing it causes a restart. Keep it trivial; don't
  check dependencies, or a database outage restarts every instance in a loop.
- **Readiness**: "can this instance serve traffic?" Failing it removes the instance from
  the load balancer.

### Reproducing safely

- Reproduce in a non-production environment with production-like data volume.
- Replay the failing request (from logs, sanitized) with the `.http` file or curl.
- Feature flags and canary deployments let you test a fix on a small slice of traffic.

---

## 7. In practice: an incident in Beacon

Let's walk through a realistic incident, end to end.

**09:40.** Alert: Beacon.Api p95 latency above 2 s, error rate 4% (normally 0.1%).

**Scope.** The RED dashboard shows only `GET /api/tickets` (the list endpoint) is slow;
`GET /api/tickets/{id}` and `POST` are normal. All instances affected. Started around
09:15, rising gradually. Errors are 504s from the load balancer.

**What changed?** No deployment today. Yesterday evening, a release added a "has unread
comments" flag to the ticket list. Monday mornings bring peak traffic.

**Hypotheses** (cheapest to check first):

1. The new flag added a slow query per ticket (N+1).
2. The database is overloaded by something else.
3. Thread pool starvation from a blocking call in the new code.

**Evidence.**

- A trace of a slow request shows **21 database spans**: one list query plus 20 small
  queries (one per ticket on the page) for unread comments. Each takes 30–80 ms under load.
  Hypothesis 1 confirmed: classic **N+1**.
- Database CPU is high, but only due to these queries (hypothesis 2 is a consequence, not a
  cause).
- `dotnet-counters` thread pool queue length is normal (hypothesis 3 rejected).

**Mitigate (09:52).** The flag is behind a feature flag (or, without one: roll back
yesterday's release). Turning it off brings p95 back to 150 ms within two minutes.

**Fix.** Load unread-comment flags for the whole page in one query (a join or `IN` list),
and add an index on `comments(ticket_id, created_at)`. Book IV covers both techniques.
Add a test that counts queries for the list endpoint (EF Core interceptors or logging can
count commands) so N+1 can't silently return.

**Learn.** A short, blameless **postmortem**:

```markdown
## Summary
GET /api/tickets p95 latency reached 6s for 37 minutes; 4% of list requests failed (504).

## Impact
Agents saw slow or failed queue loads 09:15–09:52 on Monday peak.

## Root cause
The "unread comments" feature issued one query per ticket on each page (N+1).
Under peak load, database contention made each query slow, exceeding proxy timeouts.

## What went well
Feature flag allowed mitigation in 2 minutes once identified.

## What went wrong
Load testing didn't include the list endpoint with realistic page sizes.
No alert on query count per request.

## Action items
- [ ] Batch unread-comment lookup; add index (owner: maria, due: Oct 10)
- [ ] Add query-count assertion test for list endpoints (owner: omar)
- [ ] Include list endpoints in the weekly load test (owner: platform team)
```

**Blameless** means the postmortem asks *how the system allowed this*, not *who did it*.
People who fear blame hide information, and hidden information causes the next incident.

---

## 8. What can go wrong (in the investigation itself)

- **Changing many things at once**, so you don't know what fixed it (or what broke it worse).
- **Debugging during an outage instead of mitigating.** Roll back first.
- **Confirmation bias**: committing to the first hypothesis and ignoring contrary evidence.
- **Trusting averages** instead of percentiles and distributions.
- **Missing telemetry**: discovering during the incident that the key log line doesn't
  exist. Every postmortem should ask "what evidence did we wish we had?"
- **Liveness checks that depend on the database**, turning a database blip into a restart
  storm.
- **Fixing the symptom** (restarting, adding threads, raising timeouts) and calling it done.

---

## 9. How an experienced engineer thinks about this

- **Stay calm and systematic.** Scope, check what changed, hypothesize, test cheaply.
- **Restore service first; understand second.**
- **Evidence over intuition.** Traces, metrics, dumps settle arguments.
- **Make the next incident easier**: every incident should leave better telemetry,
  better alerts and safer deploys behind.
- **Blameless culture** is an engineering practice, not a nicety.

---

## 10. Check yourself

**Questions**

1. Why mitigate before diagnosing? Give three mitigation options.
2. What questions help scope an incident?
3. What does a spike in 502 vs 504 suggest?
4. What are the symptoms of connection pool exhaustion, and its common causes?
5. What's the difference between liveness and readiness checks? Why shouldn't liveness
   check the database?
6. What is a retry storm, and how do you prevent one?
7. What makes a postmortem blameless, and why does it matter?

**Exercises**

1. Introduce an N+1 into Beacon's list endpoint deliberately, run a load test (k6 or
   `bombardier`), and find it using traces or logged SQL.
2. Introduce sync-over-async in an endpoint, load test it, and diagnose it with
   `dotnet-counters` and `dotnet-stack`.
3. Add liveness and readiness health checks to Beacon.Api.
4. Write a postmortem for a real incident you've experienced, using the template above.

**Interview-style questions**

- "Users report the API is slow. Walk me through how you'd investigate."
- "Tell me about a production incident you handled. What did you learn?"
- "How do you design an API to be easy to diagnose in production?"

---

## 11. Going deeper

- [Google SRE Book](https://sre.google/sre-book/table-of-contents/) — especially "Effective
  Troubleshooting," "Emergency Response" and "Postmortem Culture."
- [Microsoft docs: .NET diagnostics tools](https://learn.microsoft.com/dotnet/core/diagnostics/)
- [ASP.NET Core Diagnostic Scenarios](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios) by David Fowler.
- Brendan Gregg's [USE method](https://www.brendangregg.com/usemethod.html).

---

## Book III wrap-up

Beacon is now a real web API: HTTP semantics respected, a minimal API with typed
contracts, validated configuration and structured logs, a deliberate REST design with
cursor pagination, ETags and idempotency, validation in the right layers, JWT
authentication with resource-based authorization, caching where it pays off, background
workers and live updates, a security review, and a playbook for when things break.

Its data still lives in memory and vanishes on restart. That's next.

**Next:** [Book IV — SQL & PostgreSQL](../04-sql/README.md).
