# Capstone: The Complete Running Project

Beacon started in Book I as a `Ticket` class in a console application. Thirteen books later, it's a
multi-tenant support desk and knowledge base with a .NET API and BFF, a React frontend, PostgreSQL with
vector search, an event backbone, AI features with evaluation and guardrails, Python tooling, two Rust
services, and a production deployment on Azure with CI/CD, infrastructure as code and observability.

This capstone does two things. First, it **assembles** the whole system: one map of every component,
which book built it and why it exists, and two complete walkthroughs (one request, one event) that cross
every layer. Second, it runs a **production readiness review**, the structured check experienced teams do
before a system takes real customers, and works through what it finds. The review checklist is reusable
for any system you build.

---

## 1. The problem: components aren't a system

Each book taught its part in depth. But production failures rarely respect those boundaries. A slow
ticket list can be a missing index (Book IV), a re-render storm (Book VI), a cache key bug (Book III), a
connection pool limit (Book IX) or a noisy tenant (Book XIII). A security hole can be in any layer, or
in the gaps between them. Being able to hold the **whole system** in your head (how a click becomes SQL
and comes back, where every piece of data lives, what happens when each part fails) is what this book has
been building toward.

---

## 2. The mental model: Beacon in one picture

### System context

```text
                         ┌────────────────────────────────────────────────────┐
  Customers ────────────►│                                                    │──► Email provider
  (web, help assistant)  │                     BEACON                         │──► Customer webhook endpoints
  Agents & leads ───────►│  support desk + knowledge base + AI assistance     │──► AI model provider (Foundry)
  (web)                  │                                                    │◄── Identity providers
  Customer systems ─────►│  (REST API for integrations)                       │    (Entra ID, External ID)
                         └────────────────────────────────────────────────────┘
```

### Containers (deployables and data stores)

```text
 Internet
    │
 Azure Front Door + WAF (TLS, rate limits, CDN for static assets)
    │
 ┌──┴─────────────────────────── Container Apps environment (VNet, zone-redundant) ─────────────────────┐
 │                                                                                                      │
 │  ca-beacon-bff ──YARP──► ca-beacon-api ─────────────────────────────┐                                │
 │  (cookie session,        (modular monolith: Tickets, Knowledge,     │                                │
 │   OIDC, SPA host,         Directory, Notifications, Assistant,      │                                │
 │   CSRF)                   Billing; minimal APIs; SignalR)           │                                │
 │                                │                                    │                                │
 │  ca-beacon-worker ◄────────────┤ same code, worker entry point:      │                                │
 │  (outbox relay, consumers,     │ outbox, SLA monitor, indexing,      │                                │
 │   SLA monitor, indexing,       │ AI triage, notifications            │                                │
 │   AI triage)                   │                                     │                                │
 │                                │                                     │                                │
 │  beacon-relay (Rust) ◄── Service Bus "webhooks" subscription ──► customer endpoints (isolated egress) │
 │  jobs: beacon-migrate, beacon-retention                                                              │
 └──────────────────────────────────────────────────────────────────────────────────────────────────────┘
    │ private endpoints
    ├─ PostgreSQL Flexible Server (per-module schemas, RLS, pgvector, tsvector; HA standby; PgBouncer)
    ├─ Azure Managed Redis (HybridCache L2, SignalR backplane or Azure SignalR Service)
    ├─ Service Bus (topic beacon-events: notifications, search, webhooks, analytics)
    ├─ Blob Storage (attachments, exports, event payloads)
    ├─ Key Vault (remaining secrets, encryption keys)  · App Configuration (feature flags)
    └─ Microsoft Foundry (model deployments, EU region)
 Observability: OpenTelemetry → Application Insights / Log Analytics; SLO dashboards and burn-rate alerts
 Delivery: GitHub Actions (OIDC to Azure) → ACR (signed images + SBOM) → Bicep → canary → promote by digest
 Tooling: beacon-tools (Python, uv): import, export, evals, DLQ replay · beacon-logscan (Rust): log analysis
```

### Where every piece came from

| Area | Component | Built in | Why it exists |
|---|---|---|---|
| Domain | `Ticket`, `SlaRules`, `Result<T>`, domain events | Book I | Protect invariants; testable rules without I/O |
| Workflow | Git, PRs, reviews, CODEOWNERS | Book II | Safe collaboration and history that explains decisions |
| API | Minimal APIs, auth policies, ETags, Problem Details, rate limiting, caching, SignalR | Book III | A secure, well-behaved HTTP surface |
| Data | PostgreSQL schema, indexes, EF Core, outbox, RLS, roles | Book IV | Durable, consistent, queryable data |
| Frontend | React 19 + TypeScript, TanStack Query, Router, forms, a11y | Books V–VI | Usable, accessible UI with typed data |
| Security boundary | Beacon.Bff (cookie session, OIDC, YARP) | Book VI | Keep tokens out of the browser |
| Integration | Generated contracts, E2E tests, environments | Book VII | Frontend and backend that can't drift apart |
| Runtime | Linux, Nginx/TLS, hardened containers, Compose | Book VIII | Run anywhere, minimal attack surface |
| Cloud | Container Apps, managed data services, identity, networking, monitoring, CI/CD, Bicep | Book IX | Operate reliably without managing servers |
| Tooling | beacon-tools (import, export, evals) | Book X | Automation and data work in the right ecosystem |
| AI | Triage, summaries, reply drafts, RAG help assistant, incident agent, evals | Book XI | Real productivity gains, measured and safe |
| Performance | beacon-logscan, beacon-relay (Rust) | Book XII | Where throughput and isolation justified a second language |
| Architecture | Modules, event backbone, idempotency, tenant isolation, ADRs | Book XIII | Changeable, reliable, secure at scale |

---

## 3. Walkthrough 1: an agent resolves a ticket

Follow one click through the whole system. Each step names the chapter that explains it.

1. **Browser.** The agent clicks "Resolve" on T-4821. React Hook Form validates the resolution; a
   TanStack Query mutation sends `POST /api/tickets/T-4821/resolve` with `If-Match: "12"` (the ticket's
   version), an `Idempotency-Key` generated when the dialog opened, and the `X-CSRF: 1` header. The UI
   optimistically shows "Resolving…". *(Book VI, Ch 4; Book XIII, Ch 3)*
2. **Edge.** DNS resolves to Front Door, which terminates TLS, applies WAF rules and per-client rate
   limits, and forwards to the BFF over a private origin. *(Book IX, Ch 5)*
3. **BFF.** ASP.NET Core reads the `__Host-beacon` cookie, decrypts the session, checks the CSRF header,
   attaches the agent's access token, and YARP forwards to the API's internal ingress. Trace context
   (`traceparent`) propagates. *(Book VI, Ch 5; Book III, Ch 3)*
4. **API pipeline.** Middleware: forwarded headers, exception handling to Problem Details, OpenTelemetry
   span, authentication (JWT validation against Entra ID's keys), rate limiting per tenant, routing.
   *(Book III, Ch 2, 6, 9)*
5. **Endpoint filters.** The idempotency filter claims the key in PostgreSQL (`INSERT ... ON CONFLICT`).
   The authorization policy `TicketOperations.Work` runs `TicketAuthorizationHandler`: the agent must be
   on the ticket's team. The tenant context is set from the token's organization claim.
   *(Book XIII, Ch 3, 6)*
6. **Application service.** `TicketService.ResolveAsync` loads the `Ticket` aggregate through
   `ITicketRepository` (EF Core; the tenant query filter and the connection's `beacon.tenant_id` setting
   both apply; RLS enforces it in the database). It checks the ETag against `Version`.
   *(Book IV, Ch 7; Book XIII, Ch 6)*
7. **Domain.** `ticket.Resolve(resolution, now)` enforces the state machine (can't resolve a closed
   ticket), sets `ResolvedAt`, bumps `Version` to 13, and raises `TicketResolved`. *(Book I, Ch 3)*
8. **Persistence.** `SaveChangesAsync` opens one transaction: `UPDATE tickets.tickets ... WHERE id = @id
   AND version = 12` (optimistic concurrency), the `DomainEventsToOutboxInterceptor` maps `TicketResolved`
   to `TicketResolvedIntegrationEvent` and inserts an outbox row (with the current `traceparent`), and
   the idempotency record stores the response. Commit. *(Book IV, Ch 5; Book XIII, Ch 4)*
9. **Response.** `200 OK` with the updated `TicketResponse` and `ETag: "13"`. The output cache entry for
   the ticket is evicted by tag; HybridCache entries for the team's queue counts are invalidated.
   *(Book III, Ch 4, 7)*
10. **Back in the browser.** TanStack Query replaces the cached ticket with the response (read-your-writes
    without a refetch) and invalidates the list query. *(Book VI, Ch 4)*
11. **Real time.** The API publishes to the SignalR group for the team; other agents' screens update
    via `useTicketUpdates`. *(Book III, Ch 8; Book VI, Ch 3)*

Total synchronous dependencies for the agent's click: PostgreSQL. Everything else happens next.

## 4. Walkthrough 2: what happens after the commit

12. **Outbox relay** in `ca-beacon-worker` claims the row with `FOR UPDATE SKIP LOCKED`, publishes to the
    Service Bus topic `beacon-events` with `MessageId` = outbox ID, `SessionId` = ticket ID and the
    stored `traceparent`, then marks it sent. *(Book XIII, Ch 4)*
13. **Notifications subscription.** The consumer's inbox insert succeeds (first delivery), `SurveyPolicy`
    asks Billing's `IEntitlements` for the plan, a survey is created, and an email is queued through the
    Notifications outbox with an idempotency key to the provider. *(Book XIII, Ch 1, 2, 4)*
14. **Search subscription.** The indexing handler updates the ticket's `tsvector` and, if the content
    changed, re-embeds changed chunks through the embedding model, inside the tenant's scope.
    *(Book IV, Ch 3; Book XI, Ch 5)*
15. **Webhooks subscription.** `beacon-relay` (Rust) receives the event, looks up the tenant's endpoints,
    checks the URL against the SSRF guard after DNS resolution, signs the payload with the endpoint's
    secret, and POSTs with per-endpoint concurrency limits and exponential backoff.
    *(Book XII, Ch 4; Book XIII, Ch 9)*
16. **Analytics subscription.** Forwarded to the analytics pipeline; the support lead's dashboard shows
    the updated resolution time within a minute. *(Book XIII, Ch 4)*
17. **Observability.** In Application Insights, one end-to-end trace shows the click, the API request,
    the SQL statements, the outbox publish, and each consumer's processing, including the relay's
    outbound HTTP call. Metrics record resolution counts, outbox lag and end-to-end notification
    latency. *(Book IX, Ch 6; Book XIII, Ch 5)*

> **🔍 Investigation:** Try this with your own Beacon: resolve a ticket and find the single trace that
> contains steps 1–17. If any step is missing from the trace, that's an observability gap. The most
> common one is step 12, where the relay starts a new trace instead of continuing the original, because
> the `traceparent` wasn't stored in the outbox row.

---

## 5. Running the whole thing locally

A developer should be able to run all of Beacon with one command. The Aspire AppHost (Book IX)
orchestrates it:

```csharp
// src/Beacon.AppHost/AppHost.cs
var builder = DistributedApplication.CreateBuilder(args);

var postgres = builder.AddPostgres("postgres")
    .WithImage("pgvector/pgvector", "pg18")
    .WithDataVolume()
    .AddDatabase("beacon");
var redis = builder.AddRedis("redis");
var bus = builder.AddAzureServiceBus("bus").RunAsEmulator();
var events = bus.AddServiceBusTopic("beacon-events");
var mail = builder.AddContainer("mailpit", "axllent/mailpit").WithHttpEndpoint(targetPort: 8025, name: "ui");

var migrate = builder.AddProject<Projects.Beacon_Migrate>("migrate")
    .WithReference(postgres).WaitFor(postgres);

var api = builder.AddProject<Projects.Beacon_Api>("api")
    .WithReference(postgres).WithReference(redis).WithReference(bus)
    .WaitForCompletion(migrate);

builder.AddProject<Projects.Beacon_Worker>("worker")
    .WithReference(postgres).WithReference(bus).WaitForCompletion(migrate);

var bff = builder.AddProject<Projects.Beacon_Bff>("bff")
    .WithReference(api).WithExternalHttpEndpoints();

builder.AddViteApp("web", "../../web")          // Vite dev server with HMR (Aspire JavaScript hosting)
    .WithReference(bff);

builder.Build().Run();
```

`dotnet run --project src/Beacon.AppHost` starts PostgreSQL with pgvector, Redis, the Service Bus
emulator, a local mail catcher, runs migrations, then the API, worker, BFF and Vite dev server, with
the Aspire dashboard showing logs, traces and metrics for all of them. The Rust relay and the Python tools
run separately (`cargo run`, `uv run`) against the same local endpoints, or are added as executables to
the AppHost.

> **🔄 Current (as of October 2026):** Aspire's hosting APIs (resource names, emulator support, the
> JavaScript/Vite integration, `WaitForCompletion`) have evolved quickly across releases; the shape above
> is current for Aspire 13.x, but check the documentation for your version. Docker Compose (Book VIII)
> remains a perfectly good alternative, especially for teams not standardized on Aspire.

---

## 6. The production readiness review

A **production readiness review (PRR)** is a structured check, done before launch (and periodically
after), that a system can be operated safely by the people who'll be on call for it. Large engineering
organizations formalize it; small teams benefit from a lighter version. The point isn't bureaucracy. It's
to find the gaps while they're cheap to fix, rather than at 3 a.m.

Here's a checklist organized by area, with the questions that matter. It's designed to be reused for
your own systems.

### Product and requirements

- [ ] Are the target users, core flows and success metrics written down?
- [ ] Are the SLOs defined (availability, latency, freshness for async flows), agreed with stakeholders,
      and justified by user needs? *(Book IX, Ch 6)*
- [ ] Are the known limits documented (max tenants, tickets per tenant, attachment size, request rates)?

### Architecture

- [ ] Is there a current context and container diagram? *(Book XIII, Ch 2)*
- [ ] Are significant decisions recorded in ADRs?
- [ ] Are module boundaries enforced by the build?
- [ ] Is every synchronous dependency on the critical path justified, with timeouts and fallbacks?
      *(Book XIII, Ch 3)*
- [ ] Is there a dependency failure table with degraded modes?

### Code and testing

- [ ] Do unit, integration (real PostgreSQL via Testcontainers), contract and E2E tests run in CI and
      pass reliably? *(Book I, Ch 14; Book VII, Ch 3)*
- [ ] Are the critical user journeys covered by E2E tests?
- [ ] Is there a load test with production-like data, and are capacity numbers recorded?
      *(Book XIII, Ch 5)*
- [ ] Are static analysis, nullable reference types and warnings-as-errors enabled?

### Security

- [ ] Has a threat model been done for features crossing trust boundaries? *(Book XIII, Ch 6)*
- [ ] Is authorization enforced server-side on every endpoint, hub method, job and AI tool, with a
      fallback policy?
- [ ] Is tenant isolation tested by an automated cross-tenant suite, and enforced in the database?
- [ ] Are secrets eliminated (managed identity) or in Key Vault, with rotation and secret scanning?
- [ ] Are dependencies locked, scanned and updated automatically? Are actions pinned and workflows
      least-privilege? Are images signed with SBOMs?
- [ ] Are security headers, CSP, CSRF protection and cookie settings verified? *(Book III, Ch 9;
      Book VI, Ch 5)*
- [ ] Has a penetration test or external review been done for internet-facing systems?

### Data

- [ ] Are backups automatic, and **has a restore been tested** recently, with measured restore time?
      *(Book IV, Ch 8)*
- [ ] Are RPO and RTO defined and achievable?
- [ ] Are migrations backward-compatible (expand/contract), and run by the pipeline with a separate
      role?
- [ ] Is personal data inventoried, with retention and deletion implemented everywhere it goes
      (including search indexes, embeddings, logs and AI providers)?
- [ ] Are there data integrity checks for critical invariants?

### Observability and operations

- [ ] Are logs structured, free of secrets and personal data, and queryable? *(Book IX, Ch 6)*
- [ ] Do traces cover synchronous and asynchronous flows end to end?
- [ ] Are there dashboards for service health and SLOs, and alerts on symptoms (SLO burn) rather than
      causes?
- [ ] Are there runbooks for every alert, and have they been tried?
- [ ] Is there an on-call rotation with escalation, and do on-call engineers have access and training?
- [ ] Are DLQ depth, outbox lag and consumer lag alerted? *(Book XIII, Ch 4)*

### Reliability

- [ ] Is the system redundant across zones, with tested failover? *(Book IX, Ch 7)*
- [ ] Have failure scenarios been tested under load (instance loss, database failover, slow
      dependency)? *(Book XIII, Ch 5)*
- [ ] Are retries bounded, jittered and in one layer; are operations idempotent?
- [ ] Are there per-tenant limits to contain noisy neighbors?
- [ ] Is there a disaster recovery plan for regional failure, and is it documented honestly (what's lost,
      how long it takes)?

### Deployment

- [ ] Is deployment automated, from a protected branch, with approvals for production?
      *(Book IX, Ch 8)*
- [ ] Is infrastructure defined as code, with drift detection? *(Book IX, Ch 9)*
- [ ] Are releases progressive (canary) with automatic rollback on health and SLO signals?
- [ ] Can any change be rolled back in minutes, including configuration and feature flags?

### Cost

- [ ] Is there a cost model per tenant or per unit of usage, and a budget with alerts?
      *(Book IX, Ch 7)*
- [ ] Are the most expensive components (AI tokens, database, egress) measured and optimized?

### AI features

- [ ] Do AI features have evaluation datasets and quality thresholds that gate releases?
      *(Book XI, Ch 8)*
- [ ] Are prompt injection defenses, output validation and tool permissions in place? *(Book XI, Ch 9)*
- [ ] Are token budgets, rate limits and per-tenant cost limits enforced?
- [ ] Is there a human in the loop for consequential actions, and can every AI feature be switched off
      by a flag?
- [ ] Are users told when content is AI-generated, and can they give feedback?

### People and documentation

- [ ] Can a new engineer run the system locally in under an hour from the README?
- [ ] Does more than one person understand each critical component?
- [ ] Is there a support process: how customer reports reach engineering, and how incidents are
      communicated to customers?

---

## 7. In practice: Beacon's review

The team runs the review before onboarding the first large paying customers. Most items pass, because
they were built in earlier books. Here are the findings that didn't, how they were classified, and what
was done. This is what real reviews look like: a handful of serious gaps among many green checks.

| # | Area | Finding | Severity | Resolution |
|---|---|---|---|---|
| 1 | Data | Backups configured, but no restore had ever been tested. Trial restore of production-size data took 3 h 10 min, exceeding the 2 h RTO. | **Blocker** | Documented restore runbook; quarterly restore drills; moved to a larger restore target; RTO revised with stakeholders to 4 h with a plan to reduce it |
| 2 | Security | Cross-tenant test suite didn't cover the CSV export job and SignalR group joins. Export job ran without tenant context; RLS blocked it (failing closed), but the job then retried forever. | **Blocker** | Job sets tenant per export; suite generated from endpoints *and* job and hub registrations; DLQ for failed exports |
| 3 | Data | Personal data deletion didn't remove embeddings of comments from deleted users. | **Blocker** | Deletion handler now deletes chunks and embeddings; test verifies search can't find deleted content |
| 4 | Operations | 6 of 23 alerts had no runbook; two alerts fired daily and were ignored. | High | Runbooks written; noisy alerts converted to dashboard signals or tuned to SLO burn |
| 5 | Reliability | Regional disaster recovery plan existed as a diagram only. | High | Game day: restored to the paired region from geo-backups in 5 h; documented as RPO ≤ 1 h, RTO ≤ 8 h for regional loss, accepted by the business |
| 6 | AI | Help assistant evaluation dataset hadn't been updated in four months; new product areas weren't covered. Faithfulness score on a new sample: 0.81 vs 0.92 threshold. | High | Monthly dataset refresh from real (anonymized) questions; retrieval improved for new areas; release gate enforced |
| 7 | Cost | No per-tenant AI cost visibility; one trial tenant used 18% of the month's tokens. | Medium | Per-tenant token metering and limits by plan; alert at 80% of budget |
| 8 | People | Only one engineer could debug `beacon-relay` (Rust). | Medium | Pairing rotation, a runbook, and the ADR's "revisit" condition reviewed; second engineer trained |
| 9 | Architecture | Two ADRs described decisions that had since changed. | Low | Superseding ADRs written |
| 10 | Docs | Local setup took a new engineer 3 hours (Service Bus emulator configuration undocumented). | Low | AppHost updated to configure the emulator automatically; README fixed |

Blockers were fixed before launch. High items had owners and dates within the first month. Medium and low
items went into the normal backlog. The whole review took two days of the team's time and prevented at
least three incidents that would each have cost more.

> **⚠️ What can go wrong:** A readiness review done once and filed away. Systems drift: new features
> skip the cross-tenant suite, runbooks go stale, restore times grow with data. Re-run the review (or the
> parts that changed) every six to twelve months and before major launches, and automate whatever can be
> automated, such as restore drills, cross-tenant tests and checks that every alert links to a runbook.

---

## 8. Launching

With the review done, the launch itself is an exercise in limiting blast radius (Book XIII, Ch 5):

1. **Internal dogfooding**: Beacon's own support team uses it for real work for two weeks.
2. **Private beta**: 5–10 friendly customers, with a shared channel to the team, feature flags for the
   AI features, and daily review of errors, SLOs and feedback.
3. **Staged rollout**: new customers onboard in cohorts; each cohort only after the previous one's
   metrics are healthy for a week.
4. **Launch readiness on the day**: the on-call engineer and a backup are named, dashboards open, a
   rollback plan written, customer communication templates ready, no other risky changes scheduled.
5. **After launch**: a review after two weeks: what surprised us, what broke, what users actually did,
   and what to change next.

---

## 9. Day two and beyond

Launch is the beginning. Running Beacon well over years means the ongoing practices from throughout the
book:

- **Operate**: on-call, SLO reviews, blameless postmortems, capacity reviews as tenants grow.
- **Keep current**: monthly dependency updates, the yearly .NET LTS upgrade, Node and PostgreSQL major
  versions, model version changes with re-evaluation. *(Book XIII, Ch 7)*
- **Pay down debt continuously** in the hotspots, guided by delivery metrics.
- **Revisit decisions** when their review triggers fire: pgvector at 100M chunks, module extraction if a
  team needs independent deployment, a second region if customers require it.
- **Learn from users**: product analytics, support tickets about Beacon itself, AI feedback signals.
- **Keep the book current**: this book, like Beacon, is a living system. The 🔄 callouts will change;
  the 🧱 ones mostly won't.

---

## 10. What can go wrong

- **Integration gaps**: each component tested alone, but the flows between them untested and
  unobserved.
- **Treating launch as the finish line**, with no plan for operations, updates and learning.
- **Readiness theater**: checking boxes ("we have backups") without verifying them ("we restored one").
- **Big-bang launches** to all customers at once, without flags or cohorts.
- **Unowned components**: parts of the system nobody feels responsible for, typically the ones built
  in a different language or by someone who left.
- **Documentation that describes the intended system** rather than the actual one.

---

## 11. When not to use it

> **🧭 When not to use it:** Beacon is deliberately comprehensive, because it's a teaching vehicle that
> touches every topic in the book. A real product at an early stage should be **far simpler**: one .NET
> API serving the React app, PostgreSQL, a managed hosting platform, CI/CD and basic monitoring. No BFF
> until you have browser-based auth to protect, no Service Bus until in-process events and the outbox are
> outgrown, no Rust until a measured need, no AI agent until simpler AI features prove their value.
> Similarly, scale the readiness review to the stakes: a full review for a multi-tenant system holding
> customer data; a one-page checklist for an internal tool. Add each piece of Beacon's architecture when
> the problem it solves actually appears.

---

## 12. How an experienced engineer thinks about this

- **Can trace any request or event end to end**, and uses that ability to debug, review and design.
- **Knows why every component exists**, and would remove any that no longer earns its place.
- **Verifies rather than assumes**: restores backups, runs failovers, tests isolation, checks traces.
- **Launches progressively**, with flags, cohorts and rollback plans.
- **Treats operations, upgrades and learning as part of the product**, not chores after it.
- **Starts simple and grows the architecture with the problem**, using everything in this book as a
  toolbox, not a checklist to apply all at once.

---

## 13. Check yourself

**Questions**

1. Draw Beacon's container diagram from memory. Which components are on the synchronous path of
   "resolve a ticket"?
2. Walk through the 17 steps of resolving a ticket. Which steps protect against duplicates, and how?
3. Where does tenant isolation get enforced along the request path? Name at least four places.
4. What's a production readiness review for, and why is "has a restore been tested?" more important than
   "are backups enabled?"
5. Why were findings 1–3 in Beacon's review classified as blockers?
6. Describe a progressive launch plan for a new multi-tenant product.
7. Which parts of Beacon would you remove for an early-stage version of the product, and why?

**Exercises**

1. **Build Beacon end to end.** Using the chapters as your guide, get the complete system running
   locally with the AppHost, then deployed to Azure (or a Linux server with Compose). Resolve a ticket
   and find the end-to-end trace covering all 17 steps.
2. **Run the readiness review** on your Beacon (or on a system at work). Classify findings by severity
   and fix the blockers.
3. **Do a restore drill**: restore your Beacon database from backup to a new server, measure the time,
   verify data integrity, and write the runbook.
4. **Run a game day**: with a colleague playing incident commander, inject three failures (database
   failover, AI provider outage, a poison message) and handle them using only your runbooks and
   dashboards. Write a short postmortem.
5. **Simplify**: write a design for "Beacon Lite", the smallest version that serves a 20-agent team well,
   and an ADR explaining what you removed and the triggers for adding each piece back.

**Interview-style questions**

- "Walk me through what happens, end to end, when a user clicks a button in a system you built."
- "How do you know a system is ready for production?"
- "Tell me about a launch you were part of. What went wrong, and what would you do differently?"
- "How would you design the first version of a support desk product for a startup?"

---

## 14. Going deeper

- Google SRE, ["Production Readiness Review"](https://sre.google/sre-book/evolving-sre-engagement-model/)
  (in *Site Reliability Engineering*, chapter 32) — how the practice works at scale.
- Susan Fowler, *Production-Ready Microservices* — standardized readiness criteria (applicable well
  beyond microservices).
- Michael Nygard, *Release It!* — designing and launching systems that survive production.
- [Aspire documentation](https://aspire.dev) and [Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/)
  — reliability, security, cost, operations and performance checklists for Azure workloads.

**Next:** [Further Projects](02-further-projects.md) gives you new systems to design and build on your
own, each exercising a different combination of the skills in this book.
