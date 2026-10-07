# Monitoring and Observability

A system you can't see into is a system you can't operate. Book III, Chapter 10 diagnosed
incidents using logs, metrics and traces; Book VIII, Chapter 7 used Linux tools on a single
server. In the cloud, with many replicas starting and stopping, managed services you can't
SSH into, and requests crossing several components, you need **telemetry collected
centrally** and **alerts that tell you about problems before users do**.

This chapter covers observability concepts, OpenTelemetry, Azure Monitor and Application
Insights, and how to design dashboards, SLOs and alerts that are useful rather than noisy.

---

## 1. The problem: distributed systems are opaque

When a user reports "saving a comment is slow," the request crossed Front Door, the BFF, the
API, PostgreSQL, the outbox worker and SignalR, across several replicas. Questions you need
to answer quickly:

- Is it one user, one tenant, one endpoint, or everyone?
- Which component is slow?
- Did it start with a deployment?
- Is it getting worse?

Without centralized, correlated telemetry, each question means logging into a different
system, if that's even possible.

---

## 2. The mental model: monitoring vs observability

- **Monitoring** answers known questions: "is CPU above 80%?", "is the error rate above 1%?"
  You define the checks in advance.
- **Observability** is the ability to answer **new** questions from the outside, without
  deploying new code: "why are requests from tenant X with more than 50 tags slow since
  Tuesday?" It comes from rich, correlated, high-cardinality telemetry.

### The three signals (and their correlation)

| Signal | What | Strength |
|---|---|---|
| **Metrics** | Numeric time series (counts, rates, gauges, histograms) | Cheap, fast to query, ideal for dashboards and alerts |
| **Traces** | The path of one request across components, as a tree of **spans** with timings | Show *where* time goes and *which* component failed |
| **Logs** | Discrete events with context | Show *what* happened in detail |

The real power is **correlation**: a spike on a latency chart → exemplar traces from that
spike → the slow span (a SQL query) → logs from that exact request, all linked by **trace ID**.

```text
 Trace 4bf92f35… "POST /api/tickets/T-42/comments"  480 ms
 ├─ Front Door                                         8 ms
 ├─ Beacon.Bff: POST /api/...                        472 ms
 │   └─ HTTP POST api/tickets/T-42/comments          465 ms
 │       └─ Beacon.Api: POST /api/tickets/{id}/comments   462 ms
 │           ├─ PostgreSQL SELECT tickets…                 6 ms
 │           ├─ PostgreSQL SELECT comments…              418 ms   ◄── here
 │           └─ PostgreSQL INSERT / COMMIT                12 ms
 └─ (async) outbox worker → SignalR                    linked span
```

---

## 3. OpenTelemetry

**OpenTelemetry (OTel)** is the vendor-neutral standard for producing and collecting
telemetry: APIs, SDKs, semantic conventions (standard attribute names like `http.route`,
`db.system`), and the OTLP protocol. Instrument once; send to any backend (Azure Monitor,
Grafana, Datadog, Honeycomb, Jaeger, Prometheus).

.NET has OTel built in:

- **Traces** use `System.Diagnostics.ActivitySource` / `Activity` (an `Activity` is a span).
- **Metrics** use `System.Diagnostics.Metrics.Meter` (counters, histograms, gauges).
- **Logs** come from `ILogger` (Book III, Chapter 3).
- ASP.NET Core, `HttpClient`, EF Core/Npgsql, SignalR and Azure SDKs emit telemetry
  automatically when their instrumentation is enabled.

### Context propagation

For traces to cross processes, each outgoing call carries the trace context in the W3C
`traceparent` header (Book III, Chapter 1). `HttpClient` and ASP.NET Core do this
automatically; message-based flows (the outbox) must carry the context in the message so the
consumer can link its span to the producer's.

### Setting it up

```csharp
// Beacon.ServiceDefaults (a shared project referenced by Api, Bff and workers)
builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService("beacon-api", serviceVersion: builder.Configuration["BEACON_VERSION"]))
    .WithTracing(t => t
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddNpgsql()
        .AddSource("Beacon.*"))
    .WithMetrics(m => m
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddRuntimeInstrumentation()
        .AddMeter("Beacon.*"))
    .UseAzureMonitor();          // Azure Monitor OpenTelemetry Distro: exports traces, metrics, logs to Application Insights

builder.Logging.AddOpenTelemetry(o => { o.IncludeScopes = true; o.IncludeFormattedMessage = true; });
```

Locally, swap `UseAzureMonitor()` for the OTLP exporter pointing at the Aspire dashboard or
Jaeger (Book VIII, Chapter 5's `observability` profile), so you see the same telemetry while
developing.

> **🔄 Current (as of October 2026):** The **Azure Monitor OpenTelemetry Distro**
> (`Azure.Monitor.OpenTelemetry.AspNetCore`) is the recommended way to send .NET telemetry to
> Application Insights, replacing the classic Application Insights SDK. Authentication to the
> ingestion endpoint can use Entra ID (managed identity) rather than an instrumentation key.

### Custom telemetry for the domain

Platform metrics tell you the system is up; **business metrics** tell you it's working:

```csharp
// Beacon.Core/Telemetry.cs
public static class BeaconTelemetry
{
    public static readonly ActivitySource Activities = new("Beacon.Core");
    public static readonly Meter Meter = new("Beacon.Core");

    public static readonly Counter<long> TicketsCreated =
        Meter.CreateCounter<long>("beacon.tickets.created", description: "Tickets created");
    public static readonly Histogram<double> FirstResponseHours =
        Meter.CreateHistogram<double>("beacon.tickets.first_response", unit: "h");
    public static readonly Counter<long> OutboxFailures =
        Meter.CreateCounter<long>("beacon.outbox.failures");
}

// in TicketService:
BeaconTelemetry.TicketsCreated.Add(1, new KeyValuePair<string, object?>("priority", ticket.Priority.ToString()));
```

Careful with **cardinality**: tags like `priority` (4 values) are fine; tags like `ticket_id`
or `user_id` (millions of values) explode metric storage and cost. High-cardinality data
belongs in traces and logs, not metric dimensions.

---

## 4. Azure Monitor and Application Insights

**Azure Monitor** is Azure's observability platform:

- **Application Insights**: application performance monitoring (requests, dependencies,
  exceptions, traces, live metrics, application map, availability tests, user flows).
- **Log Analytics workspaces**: where logs and Application Insights data are stored and
  queried with **KQL** (Kusto Query Language).
- **Azure Monitor Metrics**: platform metrics for every Azure resource (CPU of PostgreSQL,
  replica counts of Container Apps, Front Door request counts).
- **Diagnostic settings**: route resource logs (PostgreSQL slow query logs, Key Vault access
  logs, WAF logs, Container Apps console logs) to Log Analytics.
- **Alerts** and **action groups**, **workbooks** and **dashboards**.

### KQL in practice

```kusto
// p95 latency per endpoint over the last hour, 5-minute bins
requests
| where timestamp > ago(1h) and cloud_RoleName == "beacon-api"
| summarize p95 = percentile(duration, 95), count() by name, bin(timestamp, 5m)
| render timechart

// failed requests with their exception types
requests
| where timestamp > ago(1h) and success == false
| join kind=leftouter (exceptions | project operation_Id, type, outerMessage) on operation_Id
| summarize count() by resultCode, name, type
| order by count_ desc

// slowest SQL dependencies
dependencies
| where timestamp > ago(1h) and type == "postgresql"
| summarize p95 = percentile(duration, 95), calls = count() by target, data
| top 10 by p95
```

The full end-to-end transaction view in Application Insights (search by operation ID, which
is the trace ID) shows the waterfall from section 2.

### Sampling and cost

Telemetry volume costs money (ingestion and retention). Strategies:

- **Sampling** of traces (keep a percentage, but always keep errors and slow requests:
  tail-based sampling via an OTel Collector, or rate-limited sampling in the distro).
- **Log levels**: `Information` for meaningful events only; `Debug` off in production (Book
  III, Chapter 3).
- **Retention tiers**: shorter interactive retention, cheaper long-term archive.
- **Daily caps** as a safety net (with an alert, because hitting the cap means losing data
  during an incident).

---

## 5. SLIs, SLOs and error budgets

Dashboards full of CPU graphs don't tell you whether users are happy. **Service level
objectives** do.

- **SLI** (indicator): a measurement of user experience. "Proportion of `GET /api/tickets`
  requests served successfully in under 500 ms."
- **SLO** (objective): a target for the SLI over a window. "99.5% over 28 days."
- **Error budget**: what's left: 0.5% of requests may be slow or failed. Over 28 days with 2
  million requests, that's 10,000 bad requests.

```text
 SLO 99.5%  ─────────────────────────────────────────────── budget: 10,000 bad requests / 28 d
 consumed:  ████████░░░░░░░░░░░░░░░░░░░  31% (3,100) — healthy
```

Why this matters:

- It defines "reliable enough" explicitly, with the business. 100% is neither achievable nor
  worth paying for.
- **Error budget policy**: while budget remains, ship features; when it's exhausted, prioritize
  reliability work.
- **Alerts on budget burn rate** (below) are far less noisy than alerts on raw thresholds.

Good SLIs for Beacon: availability and latency of key API endpoints, as measured at Front
Door or the BFF (closest to the user); freshness of live updates (time from commit to SignalR
delivery); outbox lag (time from event to notification sent).

---

## 6. Alerting that people trust

Bad alerting is worse than none: alerts that fire constantly get ignored, and the real one is
lost in the noise.

Principles:

1. **Alert on symptoms, not causes.** "Users are getting errors" (SLO burn) pages someone; "CPU
   is 85%" usually doesn't, because high CPU without user impact isn't an emergency.
2. **Every page must be actionable and urgent.** If the response is "wait and see," it's not a
   page; make it a ticket or a dashboard.
3. **Multi-window burn-rate alerts** for SLOs: page if the error budget is burning 14× faster
   than sustainable over both the last hour and the last 5 minutes (fast, real problems);
   create a ticket for slower burns over 6 hours.
4. **Cause-based alerts as warnings**, not pages: disk 80% full, certificate expiring in 14
   days, outbox backlog growing, database connection count high, backup failed.
5. **Runbooks**: every alert links to what to check and do (Book III, Chapter 10; Book VIII,
   Chapter 7).
6. **Review alerts** after incidents and periodically: delete or tune the noisy ones.

### Availability tests

Synthetic checks from outside (Application Insights standard tests from several regions):
`GET https://beacon.example.com/health/ready` every 5 minutes, plus a scripted journey in
staging (Book VII, Chapter 3's smoke tests). They detect outages even when no users are
active, such as at night.

---

## 7. Dashboards

A small number of focused dashboards beats hundreds of charts:

- **Service overview** (the "is it healthy?" view): SLO status and budget, request rate, error
  rate, p50/p95/p99 latency per key endpoint, active replicas, deployments marked on the
  timeline.
- **Dependencies**: PostgreSQL (CPU, connections, storage, slow queries), Redis, Service
  Bus/outbox lag, external APIs.
- **Business**: tickets created/resolved per hour, first-response time, SLA breaches.

Annotate deployments on charts so "did this start with a release?" (Book III, Chapter 10's
first question) is answerable at a glance.

---

## 8. In practice: observability for Beacon

**1. Shared defaults**: a `Beacon.ServiceDefaults` project configures OpenTelemetry (section
3), health checks and resilience defaults for all .NET services (the same pattern Aspire
templates use).

**2. Correlation through the outbox**: the outbox message stores the producer's
`traceparent`; the worker starts its span with a link to it, so a slow notification can be
traced back to the request that caused it:

```csharp
// writing (DomainEventsToOutboxInterceptor): capture Activity.Current?.Id into the message
// reading (outbox worker):
using var activity = BeaconTelemetry.Activities.StartActivity(
    "outbox.process", ActivityKind.Consumer,
    parentContext: default,
    links: message.TraceParent is { } tp && ActivityContext.TryParse(tp, null, out var ctx) ? [new ActivityLink(ctx)] : null);
```

**3. Frontend telemetry**: the SPA reports Core Web Vitals (Book VI, Chapter 6) and
unhandled errors with the same trace context where possible (the browser's request carries a
`traceparent`), so a slow interaction can be followed from the browser to the database.

**4. Resource diagnostics**: diagnostic settings send PostgreSQL logs (with
`log_min_duration_statement = 500ms`), Key Vault audit logs, Front Door/WAF logs and Container
Apps system logs to the same Log Analytics workspace.

**5. SLOs and alerts**:

| SLO | Target | Page when | Ticket when |
|---|---|---|---|
| API availability (non-5xx) at the BFF | 99.9% / 28 d | 1 h and 5 min burn rate > 14× | 6 h burn rate > 6× |
| `GET /api/tickets` latency < 500 ms | 99% / 28 d | same | same |
| Notification lag < 2 min | 99% / 28 d | — | 6 h burn rate > 6× |

Warnings (non-paging): PostgreSQL storage > 80%, connections > 80% of max, outbox backlog >
1,000 for 10 min, certificate/secret expiry < 14 days, failed backup, WAF block spike.

**6. One overview workbook** with SLO status, RED metrics per endpoint, dependency health,
replicas, and deployment annotations from the pipeline (Chapter 8).

With this, Book III, Chapter 10's incident (the N+1 query) would surface as a latency SLO
burn alert, a dashboard showing which endpoint, and traces showing 21 database spans per
request, within minutes, without anyone SSHing anywhere.

---

## 9. What can go wrong

- **Logs only**, no metrics or traces: slow, expensive investigations.
- **Uncorrelated telemetry**: missing trace context across async boundaries.
- **High-cardinality metric dimensions** exploding costs.
- **Alerting on causes** (CPU, memory) and paging for non-urgent issues: alert fatigue.
- **No SLOs**: no shared definition of "healthy."
- **Telemetry costs** growing unchecked; or daily caps silently dropping data during incidents.
- **Sensitive data in telemetry** (Book III, Chapter 3): request bodies, tokens, personal data
  in span attributes.
- **Monitoring only from inside**: no synthetic checks, so a DNS or edge outage goes unnoticed.

---

## 10. How an experienced engineer thinks about this

- **Instrument for questions you haven't thought of yet**: rich traces, structured logs,
  consistent attributes.
- **Correlate everything by trace ID**, across services and async boundaries.
- **SLOs define reliability**; error budgets turn it into decisions.
- **Page on user-facing symptoms**; everything else is a ticket or a dashboard.
- **Observability is a feature**, built and reviewed like any other.

---

## 11. Check yourself

**Questions**

1. What's the difference between monitoring and observability?
2. What does each of the three signals answer best? Why is correlation essential?
3. What is OpenTelemetry, and what does it standardize?
4. How does trace context cross service and message boundaries?
5. What is metric cardinality, and why does it matter?
6. Define SLI, SLO and error budget. Why not aim for 100%?
7. Why alert on symptoms rather than causes? What's a burn-rate alert?

**Exercises**

1. Add OpenTelemetry to Beacon.Api and Beacon.Bff, run locally with the Aspire dashboard, and
   trace a comment from the browser to PostgreSQL.
2. Propagate trace context through the outbox and verify the linked spans.
3. Write KQL queries for: top 5 slowest endpoints, error rate by status code, and requests per
   tenant.
4. Define two SLOs for an application you work on and design burn-rate alerts for them.

**Interview-style questions**

- "How would you monitor a production web application?"
- "What's distributed tracing, and how does it work?"
- "How do you avoid alert fatigue?"
- "What are SLOs and error budgets?"

---

## 12. Going deeper

- [OpenTelemetry documentation](https://opentelemetry.io/docs/) and the
  [.NET OpenTelemetry docs](https://learn.microsoft.com/dotnet/core/diagnostics/observability-with-otel)
- [Azure Monitor OpenTelemetry Distro](https://learn.microsoft.com/azure/azure-monitor/app/opentelemetry-enable)
- Google, [*Site Reliability Engineering*](https://sre.google/books/) — chapters on SLOs and
  alerting; *The Site Reliability Workbook* for burn-rate alerts.
- Charity Majors, Liz Fong-Jones, George Miranda, *Observability Engineering*.

**Next:** [Chapter 7 — Scaling, Availability and Cost](07-scaling-availability-and-cost.md)
balances reliability, performance and spend.
