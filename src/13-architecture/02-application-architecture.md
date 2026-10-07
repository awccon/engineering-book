# Application Architecture

Chapter 1 worked at the scale of classes and functions. Architecture is the same set of ideas
(cohesion, coupling, the direction of dependencies) applied to the big pieces: projects, modules,
deployables and teams. Architecture decisions are the ones that are **expensive to change later**:
how the code is divided, where data lives, what runs in which process, and which team owns what.

This chapter covers the main styles a .NET team chooses between (layered, clean/hexagonal, vertical
slices), the **modular monolith** that Beacon becomes, and microservices: the problems they actually
solve and the costs they introduce. It ends with how to make and record architecture decisions.

---

## 1. The problem: structure that fights the change you need to make

Every non-trivial system needs some structure, because no one can hold 200,000 lines in their head.
Structure lets you work on one part while safely ignoring the rest. The question is **which parts**.

Three common failures:

- **The big ball of mud**: no discernible structure. Anything calls anything, every table is read
  everywhere, and the only way to know what a change affects is to deploy it.
- **Structure by technology**: folders called `Controllers`, `Services`, `Repositories`, `DTOs`,
  `Validators`. Adding one feature touches every folder, and nothing tells you that the `Billing` service
  should never call into `Tickets` internals.
- **Distribution as structure**: splitting into microservices to impose boundaries the team couldn't
  maintain in one codebase. The boundaries are now enforced by the network, which is the most expensive
  enforcement mechanism available, and they're usually in the wrong place.

Good architecture makes the **common change local** (one feature, one module, one team) and makes
**crossing a boundary deliberate and visible**.

---

## 2. The mental model: boundaries, dependencies and deployment are separate decisions

Architecture is often discussed as one choice ("monolith or microservices?"), but it's really three
independent ones:

1. **Logical boundaries**: how the code is divided into modules, and which module may depend on which.
2. **Dependency direction**: within a module, whether business rules depend on infrastructure or the
   other way around.
3. **Physical deployment**: how many processes, how many databases, how many release pipelines.

```text
                        Logical boundaries
                     weak                strong
                  ┌───────────────────┬───────────────────┐
     one          │ Big ball of mud    │ Modular monolith  │
  Deployment      │ (most legacy apps) │ ← Beacon          │
     many         │ Distributed        │ Microservices     │
                  │ monolith (worst)   │ (done well)       │
                  └───────────────────┴───────────────────┘
```

The bottom-left corner, the **distributed monolith**, is where many microservice migrations end up:
many deployables that must be released together, share a database, and call each other synchronously in
long chains. It has all the costs of distribution and none of the benefits.

> **🧱 Durable:** Get the logical boundaries right first. Good boundaries make deployment a reversible
> choice: a well-isolated module can be extracted into a service when there's a reason. Bad boundaries
> make every deployment model painful.

---

## 3. Layered architecture

The classic: presentation → business logic → data access, each layer calling only the one below.

```text
┌──────────────────────────────┐
│ Presentation (API endpoints) │
├──────────────────────────────┤
│ Business logic (services)    │
├──────────────────────────────┤
│ Data access (EF Core, SQL)   │
└──────────────────────────────┘
          │
      Database
```

**What it gets right:** separation of HTTP concerns from business rules from SQL. Everyone understands
it. For CRUD-heavy applications with modest rules it is perfectly adequate.

**Where it struggles:**

- Business logic **depends on** data access, so the domain is shaped by the database and can't be
  tested without it (or without mocking the data layer, which tests very little).
- Layers are horizontal; features are vertical. A feature touches every layer, and the layers grow into
  huge `Services` and `Repositories` folders organized by kind rather than purpose.
- "Strict layering" produces pass-through methods (Chapter 1, section 7).

---

## 4. Clean, hexagonal and onion architecture

These three names (Alistair Cockburn's **hexagonal** / ports and adapters, Jeffrey Palermo's **onion**,
Robert Martin's **clean**) describe essentially the same idea: apply the Dependency Inversion Principle
at the architecture level, so that **the domain is at the center and depends on nothing**.

```text
                 ┌───────────────────────────────────────────┐
                 │ Infrastructure / adapters                  │
                 │  EF Core, HTTP endpoints, SignalR, OpenAI, │
                 │  email, Service Bus, file storage          │
                 │   ┌───────────────────────────────────┐   │
                 │   │ Application (use cases)            │   │
                 │   │  TicketService, SendSatisfaction…  │   │
                 │   │   ┌───────────────────────────┐   │   │
                 │   │   │ Domain                     │   │   │
                 │   │   │  Ticket, SlaRules, events, │   │   │
                 │   │   │  ports: ITicketRepository  │   │   │
                 │   │   └───────────────────────────┘   │   │
                 │   └───────────────────────────────────┘   │
                 └───────────────────────────────────────────┘
            all source dependencies point inward
```

- **Ports** are interfaces owned by the inner layers that describe what the application needs
  (`ITicketRepository`, `INotifier`, `IArticleSearch`) or offers (use-case methods).
- **Adapters** implement ports using a technology (`EfTicketRepository`) or drive the application from
  outside (minimal API endpoints, a CLI command, a message handler).

This is how Beacon has been built since Book I: `Beacon.Core` has no reference to ASP.NET Core or EF
Core; `Beacon.Infrastructure` and `Beacon.Api` reference `Beacon.Core`. The payoff has been real:
domain tests run in milliseconds without a database, and the same `TicketService` is driven by the API,
the CLI and background workers.

### Where clean architecture goes wrong

The idea is sound; the common *templates* are often heavy. Typical over-application:

- **Four or more projects per feature area** (`Domain`, `Application`, `Infrastructure`, `WebApi`,
  `Contracts`, `Shared`), each with folders for commands, queries, handlers, validators, mappers and DTOs.
  A one-field change touches eight files.
- **Abstracting the framework you'll never replace.** Wrapping EF Core in repositories that expose
  `IQueryable` (leaking it anyway), or wrapping `ILogger` in `IAppLogger`.
- **Mapping at every layer**: entity → domain model → application DTO → API response, with near-identical
  shapes. Each mapping is a place to forget a field.
- **Treating reads like writes.** Loading a full aggregate through a repository to display a list. Reads
  don't need the domain model; they need a fast projection (Book IV, Book VII).

> **🧭 When not to use it:** A CRUD service with little business logic (an admin settings API, a
> reporting endpoint, a thin integration) gains almost nothing from ports and adapters. Endpoints that
> use `DbContext` directly, with good tests against a real database, are simpler and just as
> maintainable. Reserve the full inversion for areas with **real rules worth protecting**: in Beacon,
> tickets and SLAs, not the tag list.

---

## 5. Vertical slice architecture

Jimmy Bogard's **vertical slices** organize by feature instead of layer: each request (create ticket,
resolve ticket, search articles) is a folder or file containing everything it needs, from endpoint to
SQL. Slices can share the domain model where rules matter and go straight to the database where they
don't.

```text
src/Beacon.Api/Tickets/
├── CreateTicket.cs        // request, validator, endpoint, handler: uses Ticket aggregate
├── ResolveTicket.cs
├── GetTicketDetail.cs     // query: EF Select projection straight to the response
├── ListTickets.cs         // query: raw SQL with keyset paging
└── TicketEndpoints.cs     // MapGroup wiring
```

**Strengths:** a feature change is local; each slice can choose the simplest approach for its needs; new
developers find code by feature name.

**Risks:** duplication between slices (often acceptable, see DRY in Chapter 1), and business rules
leaking into handlers if there's no domain model for the complex parts.

In practice, clean architecture and vertical slices **combine well**: a protected domain core for the
rules, vertical slices for the application layer around it. That's Beacon's shape.

---

## 6. The modular monolith

A **modular monolith** is one deployable with **strong internal boundaries**: the system is divided into
modules aligned with business capabilities, each with its own public API, private internals and its own
data, and modules communicate only through those APIs or through events.

### Finding the boundaries: bounded contexts

Domain-Driven Design's strategic idea of a **bounded context** is the best tool for finding module
boundaries. Within a context, terms have one precise meaning. Across contexts, the same word can mean
different things, and that's a hint that you've found a boundary:

- In **Ticketing**, a "customer" is the reporter of a ticket: an ID, a name, a contact preference.
- In **Billing**, a "customer" is an organization with a plan, invoices and a payment method.
- In **Identity**, there's no customer at all, just users, roles and sessions.

Forcing one `Customer` class to serve all three produces the 60-property entity every legacy system has.
Separate contexts each own their own model, linked by IDs.

Other signals for a boundary: different rates of change, different teams or stakeholders, different
consistency needs (billing must be exact; analytics can lag), and different scaling or security
requirements.

### Beacon's modules

```text
Beacon (one ASP.NET Core host, one PostgreSQL server)
├── Tickets        tickets, comments, SLA, assignment, state machine        schema: tickets
├── Knowledge      articles, versions, search (tsvector + pgvector)         schema: knowledge
├── Directory      users, teams, organizations, preferences (from IdP)      schema: directory
├── Notifications  channels, templates, delivery log; consumes events       schema: notifications
├── Assistant      AI triage, summaries, help assistant, agent              schema: assistant
└── Billing        plans, entitlements (read by others through an API)     schema: billing
```

Each module is a project (or a folder with enforced rules) with three kinds of code:

```text
src/Modules/Tickets/
├── Beacon.Tickets.Contracts/     // public: ITicketsModule, TicketResolved (integration event), DTOs
└── Beacon.Tickets/               // internal: domain, EF config, endpoints, handlers
    ├── Domain/                   // Ticket, SlaRules: pure
    ├── Features/                 // vertical slices
    ├── Persistence/              // TicketsDbContext (schema "tickets")
    └── TicketsModule.cs          // AddTicketsModule(), MapTicketsEndpoints()
```

The rules:

1. **Other modules reference only `*.Contracts`**, never the implementation project. In .NET, make
   everything in the implementation `internal` by default; the compiler then enforces most of the
   boundary.
2. **Each module owns its tables.** No module queries another module's schema. Use separate PostgreSQL
   schemas and, ideally, separate database roles so the database enforces it too.
3. **Cross-module communication** is either a synchronous call through the public contract (for queries
   that need an answer now) or an **integration event** through the outbox (for "something happened").
4. **No shared domain model.** Modules share primitive IDs (`TicketId`, `UserId`) and contracts, not
   entities.

```csharp
// Beacon.Billing.Contracts — what other modules may ask Billing
public interface IEntitlements
{
    ValueTask<Plan> GetPlanAsync(OrganizationId org, CancellationToken ct);
    ValueTask<bool> HasFeatureAsync(OrganizationId org, Feature feature, CancellationToken ct);
}

// Beacon.Tickets.Contracts — what Tickets announces
public sealed record TicketResolvedIntegrationEvent(
    Guid EventId, string TicketId, string ReporterId, string Resolution, DateTimeOffset OccurredAt);
```

Notice the integration event uses primitive types and an explicit `EventId`. Unlike the internal domain
event, it's a **published contract**: once another module (or later another service) consumes it, its
shape is versioned like an API (Book VII, Chapter 2; Chapter 4 of this book).

### Enforcing boundaries

Boundaries that aren't enforced erode within months. Layers of enforcement, cheapest first:

- **Project references** and `internal` visibility: the compiler refuses most violations.
- **Architecture tests** that fail the build on forbidden dependencies:

```csharp
public sealed class ModuleBoundaryTests
{
    private static readonly string[] Modules = ["Tickets", "Knowledge", "Directory", "Notifications", "Assistant", "Billing"];

    [Theory]
    [MemberData(nameof(ModulePairs))]
    public void Modules_depend_only_on_each_others_contracts(string module, string other)
    {
        var assembly = Assembly.Load($"Beacon.{module}");
        var referenced = assembly.GetReferencedAssemblies().Select(a => a.Name);

        Assert.DoesNotContain($"Beacon.{other}", referenced);   // use Beacon.{other}.Contracts instead
    }

    public static TheoryData<string, string> ModulePairs()
    {
        var data = new TheoryData<string, string>();
        foreach (var a in Modules) foreach (var b in Modules) if (a != b) data.Add(a, b);
        return data;
    }
}
```

  Libraries such as NetArchTest and ArchUnitNET express richer rules (namespaces, attributes, "domain
  must not reference EF Core").
- **Database roles** per module, so a stray cross-schema query fails in development rather than in a code
  review.
- **Code ownership** (`CODEOWNERS`, Book II) so changes to a module's contracts need its owners' review.

### Why the modular monolith is the default

For most teams and most systems, it combines the benefits people want from microservices (clear
ownership, independent reasoning, enforced boundaries) with the operational simplicity of one
deployable:

- One build, one deployment, one process to debug, one set of logs.
- In-process calls: no network failures, no serialization, no partial failures between modules.
- **Transactions are still available** when you really need them, and the outbox gives you reliable
  events when you don't.
- Refactoring across modules is a compiler-checked change, not a coordinated multi-service release.
- Extraction remains possible: a module with its own schema, a contracts project and event-based
  communication is most of the way to being a service.

> **🔄 Current (as of October 2026):** .NET's tooling supports this style well: Aspire (previously ".NET
> Aspire"; the AppHost and `ServiceDefaults` project from Book IX) orchestrates local development and
> standardizes telemetry whether you run one process or ten, and the `internal`-by-default plus
> architecture-test approach needs no special framework. Several "modular monolith" templates exist on
> GitHub; treat them as examples, not requirements.

---

## 7. Microservices: problems solved and costs introduced

A **microservice** is an independently deployable service that owns its data and a business capability,
communicating with others over the network. The architecture became popular through organizations like
Amazon and Netflix, whose problems were real. The question is whether you have the same problems.

### What microservices actually solve

1. **Independent deployment for many teams.** With 30 teams in one deployable, release coordination
   becomes the bottleneck. Separate services let each team ship on its own schedule. This is the
   primary benefit, and it's an **organizational** benefit.
2. **Independent scaling.** A component with very different load (image processing, search indexing,
   webhook delivery) can scale separately. Often, though, a scaled-out monolith or a separate worker
   process gets you this too.
3. **Fault isolation.** A memory leak in the report generator doesn't take down ticket creation. Only
   true if the services don't depend on each other synchronously.
4. **Technology fit.** A component that benefits from a different language or runtime: Beacon's
   `beacon-relay` in Rust (Book XII) and `beacon-tools` in Python (Book X).
5. **Security isolation.** A component with sensitive data or risky inputs (payment processing, a
   sandbox that runs customer-supplied code) gets its own boundary, identity and network rules.

### What they cost

| Cost | What it means in practice |
|---|---|
| **Network calls** | Latency, timeouts, retries, partial failure on every interaction (Chapter 3) |
| **No transactions across services** | Sagas, compensation, eventual consistency (Chapter 4) |
| **Data duplication** | Each service keeps its own copy of what it needs, kept in sync by events |
| **Distributed debugging** | Traces, correlation IDs and centralized logs become mandatory (Chapter 5) |
| **Contract versioning** | Every interface is a public API that must stay backward compatible |
| **Operational overhead** | N pipelines, N dashboards, N on-call runbooks, N sets of dependencies to patch |
| **Testing** | Contract tests and environments with many services; E2E tests get slow and flaky |
| **Local development** | Running 15 services on a laptop, or mocking most of them |
| **Refactoring boundaries** | Moving a responsibility between services is a migration, not a refactor |

Martin Fowler's "Microservice Premium" summarizes it: microservices carry a fixed cost that only pays off
above a certain level of system and organizational complexity. Below it, they slow you down.

> **🧱 Durable:** **Conway's Law**: systems mirror the communication structure of the organizations that
> build them. Service boundaries that don't match team boundaries produce constant cross-team
> coordination. Teams sometimes apply the "inverse Conway maneuver", organizing teams around the
> architecture they want. Either way, architecture and team design are the same conversation.

### When to extract a service

Beacon remains a modular monolith with a few satellite processes. Each satellite exists for a stated
reason:

| Component | Why separate | Communication |
|---|---|---|
| `Beacon.Bff` | Security boundary for the browser (Book VI, Chapter 5) | HTTP proxy |
| `ca-beacon-worker` | Background load isolated from request latency (Book IX) | Same code, same DB, different process |
| `beacon-relay` (Rust) | High-concurrency outbound HTTP; untrusted endpoints isolated | Reads outbox / HTTP intake |
| `beacon-tools` (Python) | Data and AI tooling in the ecosystem built for it | Calls the public API |
| `beacon-logscan` (Rust) | Offline CLI | Files |

Note that the worker is the **same codebase deployed twice** with different entry points: a separate
process for operational reasons without a separate service boundary. That's often all you need.

A good checklist before extracting a module into a service:

1. Is there a **concrete** reason from the list in "What microservices actually solve"? Name it.
2. Is the module **already well-bounded** in the monolith (own schema, contracts, events)? If not, fix
   that first; extraction won't create boundaries, it will just make bad ones expensive.
3. Can it work when its dependencies are **down or slow**? If every request needs a synchronous call to
   the monolith, you've created a distributed monolith.
4. Do you have the **operational basics**: tracing, centralized logs, per-service dashboards and alerts,
   automated deployment (Book IX)?
5. Who **owns** it, on call included?

### The strangler pattern in the other direction

Many teams today are going the opposite way: **merging** microservices back into modular monoliths after
paying the costs without enough benefit. Some well-publicized examples (such as Amazon Prime Video's
monitoring service in 2023, and Segment's earlier "goodbye microservices") described large cost and
complexity reductions. The lesson isn't that microservices are wrong; it's that the decision should be
driven by concrete forces, and is reversible only if the boundaries are good.

---

## 8. In practice: turning Beacon into a modular monolith

Up to now, Beacon has had one `Beacon.Core` and one `Beacon.Infrastructure`. As features grew (knowledge
base, notifications, AI assistant), those projects started to show the "structure by technology"
problem: `Beacon.Infrastructure/Persistence` has configurations for every table, and `BeaconDbContext`
has 25 `DbSet`s. Here's how to restructure incrementally, without a big-bang rewrite.

**Step 1: Map the contexts.** Write down the modules and, for each, the tables it owns and the other
modules it calls. Use real data: grep for `DbSet` usage and cross-feature calls. This produces a
dependency matrix; cycles in it are the first things to fix.

**Step 2: Create module folders inside existing projects** and move code feature by feature. Make types
`internal` as you go. Don't change behavior in the same pull request (Book II, Chapter 3: refactoring
and behavior changes in separate PRs).

**Step 3: Split the `DbContext`.** Each module gets its own context, mapped to its own schema:

```csharp
internal sealed class TicketsDbContext(DbContextOptions<TicketsDbContext> options) : DbContext(options)
{
    public DbSet<Ticket> Tickets => Set<Ticket>();

    protected override void OnModelCreating(ModelBuilder model)
    {
        model.HasDefaultSchema("tickets");
        model.ApplyConfigurationsFromAssembly(typeof(TicketsDbContext).Assembly,
            t => t.Namespace?.StartsWith("Beacon.Tickets") == true);
    }
}
```

Moving tables between schemas is an `ALTER TABLE ... SET SCHEMA` migration (cheap metadata change in
PostgreSQL), done one module at a time, with a compatibility view in the old location if reporting
queries still use it (Book IV, Chapter 8: expand/contract).

**Step 4: Replace cross-module joins.** The tricky part. A ticket list that shows the reporter's
organization name used to join `tickets` to `users` to `organizations`. Options:

- **Compose in the application**: query tickets, then batch-fetch names through `IDirectory` (one extra
  query, not N).
- **Keep a local read model**: Tickets keeps `reporter_display_name` updated from Directory's
  `UserRenamed` events. Fast reads, eventual consistency (Chapter 4).
- **A reporting read model** owned by no single module, built from events or by a read-only database role
  across schemas: acceptable for analytics, never for writes.

**Step 5: Replace cross-module calls into internals** with contract interfaces or integration events.
The survey handler from Chapter 1 becomes a handler in **Notifications** that consumes Tickets'
`TicketResolvedIntegrationEvent` and asks Billing's `IEntitlements` for the plan.

**Step 6: Add the architecture tests** and turn on database roles per schema. From now on, the boundaries
are enforced.

```text
Before                                   After
Beacon.Core ──────────┐                  Beacon.Tickets ─────► Beacon.Billing.Contracts
Beacon.Infrastructure ┤ everything       Beacon.Notifications ► Beacon.Tickets.Contracts (events)
Beacon.Api ───────────┘ sees everything  Beacon.Assistant ───► Beacon.Knowledge.Contracts
                                         Beacon.Api (host) composes all modules
```

The host becomes thin: it calls `AddTicketsModule()`, `AddKnowledgeModule()` and so on, maps their
endpoints, and configures cross-cutting concerns (authentication, telemetry, rate limiting).

> **⚠️ What can go wrong:** The most common mistake in this migration is splitting by entity rather than
> capability (a "Users module", a "Comments module"). Comments have no meaning without tickets; they
> belong to the Tickets module. If two candidate modules need to change together for most features, or
> call each other constantly, they're one module.

---

## 9. Making and recording architecture decisions

Architecture decisions outlive the people who made them. A new engineer who sees PostgreSQL schemas per
module, or a Rust webhook relay, needs to know **why**, or they'll either cargo-cult it or undo it.

**Architecture Decision Records (ADRs)** are short Markdown files in the repository, one per significant
decision:

```markdown
# ADR 0014: Deliver webhooks from a separate Rust service

- Status: Accepted (2026-09-02)
- Deciders: platform team, tickets team

## Context
Webhook volume is ~40/s at peak with bursts to 2,000/s during bulk imports. Customer endpoints are
slow (p95 2.1 s) and some time out. Delivering from ca-beacon-worker caused thread-pool pressure that
delayed SLA monitoring and indexing. Endpoints are customer-controlled URLs (SSRF risk).

## Decision
Deliver webhooks from beacon-relay, a separate Rust service that reads the outbox, with per-endpoint
concurrency limits and signed requests. It has no access to Beacon's database other than the outbox
table (dedicated role), and egress only to the public internet.

## Consequences
+ Delivery bursts no longer affect other background work; blast radius of SSRF bugs reduced.
- A second language and toolchain to maintain; two engineers currently know Rust.
- New dashboards and alerts for the relay.
Alternatives considered: a separate .NET worker process (rejected), Azure Functions with Service Bus
(rejected: cost at burst volume).
```

Good ADRs are **short**, state the **context and forces** (numbers help), list **alternatives
considered**, and are honest about **negative consequences**. They're never edited after acceptance;
a new ADR supersedes an old one, preserving the history of thinking.

A few other lightweight tools:

- **C4 diagrams** (Simon Brown): Context (the system and its users and neighbors), Containers
  (deployables and data stores), Components (modules inside a container). Two or three diagrams at the
  first two levels cover most needs. Keep them in the repo as code (Mermaid, Structurizr) so they're
  reviewed with changes.
- **Fitness functions**: automated checks that an architectural property holds: the boundary tests
  above, a performance test that fails if p95 regresses, a dependency check that fails on a forbidden
  package.
- **Reversibility**: classify decisions as one-way doors (database choice, public API shape, event
  contracts) and two-way doors (an internal library, a module's folder structure). Spend analysis time
  on the one-way doors and decide the others quickly.

> **🔍 Investigation:** The ADR above has a weak spot: the first rejected alternative has no reason.
> In a real review you'd ask: "Why not a separate .NET worker with the same isolation?" If the honest
> answer is "we wanted to try Rust", say so; that's a legitimate reason if the team accepts the cost,
> but it should be recorded as the reason. Book XII, Chapter 1 discussed when Rust's advantages justify a
> second language.

---

## 10. What can go wrong

- **Architecture astronautics.** Designing for scale, teams and flexibility you don't have. Start with
  the simplest structure that keeps boundaries clear; earn complexity.
- **Shared database between services.** The fastest way to a distributed monolith: services coupled
  through table schemas, with no owner able to change them.
- **Synchronous chains.** Service A calls B calls C calls D for every request. Availability multiplies
  (four services at 99.9% each give at most ~99.6%), latency adds, and one slow service stalls all.
- **Entity services.** `UserService`, `TicketService`, `CommentService` as separate deployables: every
  feature touches all of them. Services should own capabilities, not tables.
- **Boundaries without enforcement.** A diagram on a wiki that the code ignores. If the build doesn't
  fail, the boundary doesn't exist.
- **Common library sprawl.** A shared NuGet package used by all services that contains domain types.
  Every change requires updating every service: a distributed monolith through a package manager.
- **Big-bang rewrites.** Replacing a working system with a new architecture all at once. Chapter 7 covers
  the incremental alternative.
- **No owner.** Modules or services that "everyone" maintains are maintained by no one.

---

## 11. When not to use it

> **🧭 When not to use it:**
> - **Microservices** for a single team, a new product without proven boundaries, or a system without
>   strong operational maturity. Start with a modular monolith.
> - **Full clean architecture** for CRUD, small tools and short-lived services. Use direct code with good
>   tests.
> - **Formal module boundaries** in a very small application (a few thousand lines, one or two
>   developers). Folders by feature are enough until the codebase or team grows.
> - **ADRs for every choice.** Record decisions that are expensive to reverse or likely to be
>   questioned. A team that writes ADRs for logging library upgrades stops reading them.

---

## 12. How an experienced engineer thinks about this

- **Separates the three decisions**: logical boundaries, dependency direction, deployment. Most debates
  improve once people notice which one they're arguing about.
- **Draws boundaries around business capabilities and language**, not around entities or technical
  layers.
- **Defaults to a modular monolith** and extracts services for named, concrete reasons, recorded in an
  ADR.
- **Enforces boundaries automatically**: compiler visibility, architecture tests, database roles.
- **Treats architecture as organizational design.** Team structure, ownership and on-call are part of the
  architecture.
- **Keeps decisions reversible** where possible, and spends deliberation on the irreversible ones: data
  ownership and published contracts.
- **Uses the simplest style per area**: protected domain where the rules are, direct data access where
  they aren't.
- **Distrusts architecture diagrams that aren't backed by code**, and asks to see the dependency graph.

---

## 13. Check yourself

**Questions**

1. What three decisions are often confused in "monolith vs microservices" debates?
2. What's a distributed monolith, and how do teams end up with one?
3. How does clean/hexagonal architecture apply the Dependency Inversion Principle? What are ports and
   adapters?
4. List three ways clean architecture templates are commonly over-applied.
5. What's a bounded context, and what signals suggest a boundary?
6. What four rules keep a modular monolith's modules independent? How are they enforced?
7. Name the five problems microservices solve. Which is the most important, and why is it
   organizational?
8. What's Conway's Law, and how does it affect service boundaries?
9. What belongs in an ADR?

**Exercises**

1. Draw a C4 Context and Container diagram for Beacon as it stands at the end of Book XII.
2. Build the module dependency matrix for a codebase you know. Where are the cycles? Which "modules"
   always change together?
3. Implement the architecture tests from section 6 for Beacon, plus a rule that no `*.Domain` namespace
   references `Microsoft.EntityFrameworkCore`.
4. Move the Knowledge module's tables into a `knowledge` schema with an expand/contract migration and a
   module-specific database role. Verify that a cross-schema query from the Tickets role fails.
5. Write an ADR for a decision in a project you work on that currently has no written rationale.

**Interview-style questions**

- "Would you start a new product with microservices? Why or why not?"
- "How do you decide where to draw service or module boundaries?"
- "What's the difference between clean architecture and vertical slice architecture? Can they coexist?"
- "Tell me about a time an architectural decision turned out to be wrong. How did you change it?"
- "How do two services share data without sharing a database?"

---

## 14. Going deeper

- Sam Newman, *Building Microservices* (2nd ed.) and *Monolith to Microservices* — balanced, practical,
  including when not to.
- Eric Evans, *Domain-Driven Design*, Part IV (strategic design), and Vlad Khononov, *Learning
  Domain-Driven Design* — bounded contexts and context mapping.
- Neal Ford, Mark Richards et al., *Software Architecture: The Hard Parts* — trade-off analysis for
  distributed architectures.
- Matthew Skelton and Manuel Pais, *Team Topologies* — architecture and team structure together.
- Simon Brown, [The C4 model](https://c4model.com).
- Michael Nygard, ["Documenting Architecture Decisions"](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)
  — the original ADR post.
- Martin Fowler, ["MonolithFirst"](https://martinfowler.com/bliki/MonolithFirst.html) and
  ["Microservice Premium"](https://martinfowler.com/bliki/MicroservicePremium.html).

**Next:** [Distributed Systems](03-distributed-systems.md) looks at what changes the moment two
processes talk over a network: partial failure, time, consistency, retries and idempotency.
