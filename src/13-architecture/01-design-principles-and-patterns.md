# Design Principles and Patterns

Every book so far has made design decisions along the way: a rich `Ticket` that protects its own
invariants (Book I, Chapter 3), dependencies passed in through constructors (Book I, Chapter 13), an
outbox instead of a direct publish (Book IV, Chapter 5), a decorator stack around the chat client
(Book XI, Chapter 10). This chapter names the principles behind those decisions, looks honestly at the
famous ones (SOLID, the Gang of Four patterns, DRY), and spends as much time on **when they hurt** as
on when they help.

The goal isn't to memorize a catalog. It's to recognize the *forces* in a piece of code (what changes,
what must stay correct, who depends on whom) and to pick the simplest structure that balances them.

---

## 1. The problem: code that is hard to change

Most code is read and changed far more often than it is written. The cost of software over its life is
dominated by change: new features, fixed bugs, new regulations, new integrations. Design is the
discipline of making the *likely* changes cheap and the *dangerous* changes hard.

Two symptoms tell you a design is failing:

- **Shotgun surgery**: one conceptual change ("add a priority level") touches fifteen files in six
  projects.
- **Fragility**: a change in one place breaks something apparently unrelated ("we changed how SLA is
  computed and the CSV export broke").

And one symptom tells you a design has been *over*-applied:

- **Indirection without payoff**: to understand what happens when a ticket is created you open an
  interface, its factory, a handler, a pipeline behavior, a mapper profile and a specification, and
  none of them has a second implementation.

Both failures are expensive. The first makes change risky; the second makes every change slow and every
reader confused. Good design sits between them, and where exactly depends on how much the code is
likely to change.

---

## 2. The mental model: coupling, cohesion and the direction of change

Almost every design principle is a special case of two ideas from the 1970s:

- **Cohesion**: things that change together should live together. A module is cohesive when its parts
  serve one purpose and change for the same reasons.
- **Coupling**: things that change independently should not depend on each other's details. Two modules
  are coupled when a change to one forces a change to the other.

You want **high cohesion and low coupling**. But "low coupling" doesn't mean "no dependencies"; code
that depends on nothing does nothing. It means depending on things that are **more stable than you
are**, through the **narrowest** interface that does the job.

> **🧱 Durable:** Ask of every dependency: *which way does change flow?* If A depends on B, changes in B
> can break A. So A should depend on B only if B changes less often than A, or if the dependency is
> through a small, deliberate contract. Most principles in this chapter are ways of pointing
> dependencies toward stable things.

Some useful kinds of coupling, from loosest to tightest:

| Coupling | Example in Beacon | Cost of change |
|---|---|---|
| Data (a message or contract) | Webhook payload to `beacon-relay` | Version the contract |
| Interface | `TicketService` → `ITicketRepository` | Change the interface (rare) |
| Concrete class | `TicketEndpoints` → `TicketService` | Recompile, usually fine in one codebase |
| Internal details | Reporting SQL reads `tickets.custom_fields` JSON shape | Any refactor may break it |
| Temporal | "Call `Init()` before `Run()`" | Invisible until it fails |
| Shared mutable state | Two services writing the same table | Unbounded |

Notice that concrete coupling inside one deployable is often **fine**. The compiler finds every
caller, the refactoring tool updates them, and the tests run in one go. Coupling becomes expensive when
it crosses a boundary the compiler can't see: a network call, a database shared between teams, a
public package, a file format.

### Essential and accidental complexity

Fred Brooks distinguished **essential complexity** (inherent in the problem: SLA rules really do vary by
priority, plan and business hours) from **accidental complexity** (introduced by our tools and choices:
mapping layers, framework ceremony, a pattern applied where none was needed). Design can't remove
essential complexity; it can only put it in one place where it's visible and tested. It can, and should,
minimize accidental complexity.

A good heuristic: **every abstraction must pay rent.** It earns its place by isolating a real source of
change, enabling a test you need, or hiding complexity the caller genuinely doesn't care about. If you
can't say which, inline it.

---

## 3. SOLID, honestly

SOLID is five principles collected by Robert C. Martin in the early 2000s. They are useful, often
misapplied, and best understood by the problem each one solves.

### S — Single Responsibility Principle

> A module should have one reason to change. (Later restated: one *actor* it serves.)

The useful reading is about **people and forces**, not about "doing one thing". A class that computes
SLA deadlines *and* formats them for the email template will change when the support lead changes the
SLA policy *and* when marketing changes the email design. Those are two actors; split them.

```csharp
// Before: two reasons to change in one place
public sealed class SlaReport
{
    public string Render(Ticket t, DateTimeOffset now)
    {
        var target = t.Priority switch
        {
            TicketPriority.Urgent => TimeSpan.FromHours(1),
            TicketPriority.High => TimeSpan.FromHours(4),
            _ => TimeSpan.FromHours(24),
        };
        var due = t.CreatedAt + target;
        return $"<b>{t.Id}</b> due {due:yyyy-MM-dd HH:mm} ({(due < now ? "BREACHED" : "ok")})";
    }
}

// After: policy (support lead's rules) and presentation (email design) change independently
public static class SlaRules           // already exists in Beacon.Core.Tickets (Book I)
{
    public static TimeSpan ResponseTarget(TicketPriority p) => p switch { /* ... */ };
    public static bool IsBreaching(Ticket t, DateTimeOffset now) => /* ... */;
}

public sealed class SlaEmailFormatter
{
    public string Render(Ticket t, DateTimeOffset now) =>
        $"<b>{t.Id}</b> due {t.CreatedAt + SlaRules.ResponseTarget(t.Priority):yyyy-MM-dd HH:mm} " +
        $"({(SlaRules.IsBreaching(t, now) ? "BREACHED" : "ok")})";
}
```

The misreading is "every class does exactly one tiny thing", which produces `TicketTitleValidator`,
`TicketTitleTrimmer` and `TicketTitleLengthChecker`: three classes that always change together and
should be one method. **Cohesion is the other half of SRP**: don't split things that change for the
same reason.

### O — Open/Closed Principle

> Software entities should be open for extension but closed for modification.

The idea: when a new *variant* appears, you add code rather than editing working code. Beacon's
notification channels are the textbook case. `INotifier` (Book I) has email, in-app and webhook
implementations; adding Microsoft Teams is a new class plus a DI registration, and nothing that sends
notifications changes.

The honest caveat: you can only be "closed" against changes you **predicted**. OCP applied
speculatively ("what if we need another database?") creates extension points nobody uses. Apply it
**after the second or third variant appears**, or where variants are the business (payment providers,
notification channels, AI models). Elsewhere, editing the `switch` is fine. A `switch` over a closed
enum with exhaustiveness checking (C# switch expressions warn on missing cases; Rust makes them an
error) is often *clearer* than a polymorphic hierarchy.

### L — Liskov Substitution Principle

> Subtypes must be usable wherever their base type is expected, without the caller knowing.

This one is not a matter of taste; violating it causes bugs. If `ReadOnlyTicketRepository :
ITicketRepository` throws `NotSupportedException` from `SaveAsync`, every caller that trusted the
interface is now wrong. The contract includes **behavior**, not just signatures: preconditions (a
subtype may not demand more), postconditions (a subtype may not promise less), invariants and
exceptions.

The .NET base library has famous violations: arrays implement `IList<T>` but throw on `Add`, and
`ReadOnlyCollection<T>` does the same. That's why `IReadOnlyList<T>` exists. The fix is almost always
**a narrower interface** rather than a subtype that refuses half of the contract.

> **⚠️ What can go wrong:** LSP violations hide in fakes. An in-memory repository that doesn't enforce the
> unique constraint, doesn't bump `Version`, or returns the same object instance (so mutations "save"
> without calling `SaveAsync`) lets tests pass that fail against PostgreSQL. This is why Book IV moved
> Beacon's integration tests to Testcontainers.

### I — Interface Segregation Principle

> Clients should not be forced to depend on methods they don't use.

A 30-method `ITicketStore` forces every consumer, and every fake, to know about all 30. Split by
**client need**: the SLA monitor needs "open tickets due before X"; the API needs find and save; the
reporting job needs read-only projections. Small, role-based interfaces are easy to fake, easy to
understand, and make dependencies honest.

But don't create an interface per class "for testability" when the class has no I/O and one
implementation. `SlaRules` is pure; test it directly. Interfaces are for **boundaries**: I/O, external
services, time (`TimeProvider`), and genuine variation.

### D — Dependency Inversion Principle

> High-level policy should not depend on low-level detail; both should depend on abstractions, and the
> abstraction belongs to the high-level side.

This is the most important of the five for architecture. `TicketService` (policy) needs to load and save
tickets. If it referenced `EfTicketRepository` directly, the domain would depend on EF Core and
PostgreSQL: a change of persistence detail would ripple into business rules, and the rules couldn't be
tested without a database. Instead `Beacon.Core` **owns** `ITicketRepository`, phrased in domain terms,
and `Beacon.Infrastructure` implements it:

```text
Beacon.Core                         Beacon.Infrastructure
┌──────────────────────────┐        ┌──────────────────────────┐
│ TicketService            │        │ EfTicketRepository       │
│   uses ITicketRepository │◄───────│   implements             │
│ ITicketRepository (owned)│  refs  │   ITicketRepository      │
└──────────────────────────┘        └──────────────────────────┘
        source dependency points *toward* the policy
```

The dependency arrow in the source code points opposite to the flow of control at run time. That's the
"inversion". Chapter 2 builds whole architectures (hexagonal, clean) on this one idea.

Note what DIP is **not**: it isn't "use a DI container", and it isn't "every class needs an interface".
It's about who owns the contract. An interface that lives next to its only implementation in the
infrastructure project, shaped like EF Core's API, inverts nothing.

### SOLID in one sentence

Put things that change together in one place, and make dependencies point toward the things that
change least, through contracts the dependent side owns. Everything else is detail and judgment.

---

## 4. The other principles that matter

### DRY, and its trap

"Don't Repeat Yourself" originally (*The Pragmatic Programmer*) meant: every piece of **knowledge** has
one authoritative representation. The SLA policy should exist once, not in the API, the frontend and a
SQL view, each slightly different.

It does **not** mean "no two pieces of code may look alike". Two validation rules that happen to both
check `Length <= 200` today represent different knowledge (ticket titles and article slugs) and will
diverge. Merging them couples two things that change for different reasons, and the shared helper grows
flags: `Validate(text, isSlug: true, allowEmpty: false, legacyMode: true)`.

> **🧱 Durable:** **Duplication is far cheaper than the wrong abstraction** (Sandi Metz). Prefer to
> tolerate duplication until you understand what varies; the "rule of three" (abstract on the third
> occurrence) is a good default. When a shared abstraction has grown flags and conditionals for each
> caller, inline it back into the callers and start again.

Beacon's contract generation (Book VII, Chapter 2) is DRY done right: the API's OpenAPI document is the
single source of truth for request and response shapes, and the TypeScript types are *generated*, not
hand-copied.

### KISS and YAGNI

- **KISS** (keep it simple): prefer the solution with fewer moving parts, fewer concepts and less
  indirection, as long as it meets the requirement.
- **YAGNI** (you aren't gonna need it): don't build for requirements you don't have. Every speculative
  feature costs building, testing, documentation, and the cognitive load of everyone who reads it.

YAGNI applies to **features and flexibility**, not to **quality**. Tests, error handling, logging,
security and the ability to change the code later are not speculative. The skill is telling them apart:
"support multiple databases" is speculative; "don't put SQL strings in the controller" keeps the code
changeable for the change you *will* make.

### Make illegal states unrepresentable

Encode rules in types so that wrong code doesn't compile. Beacon already does this in several places:

- `TicketId` is a `readonly record struct`, not an `int`, so you can't pass an article ID where a
  ticket ID is expected.
- `Ticket` exposes `Resolve()` and `Close()` rather than a settable `Status`, so the state machine can't
  be bypassed.
- The frontend's branded `TicketId` type and `as const` status arrays (Book V) do the same in
  TypeScript.
- Rust's enums with data (Book XII, Chapter 3) take it furthest: a `Delivery` that is `Pending`,
  `Delivered { at }` or `Failed { reason }` can't be "delivered with a failure reason".

A rule enforced by the type system is checked on every build, by every developer, forever. A rule
enforced by a comment is checked by nobody.

### Composition over inheritance

Covered in Book I, Chapter 3: inheritance couples a subclass to its base's implementation details, and
deep hierarchies make behavior hard to locate. Composing small objects (a notifier that *has* a
formatter and a transport, rather than `HtmlEmailNotifier : EmailNotifier : NotifierBase`) keeps each
piece replaceable and testable. Use inheritance for genuine "is-a" relationships with a stable base, and
for framework extension points that require it (`BackgroundService`, `DbContext`).

### Tell, don't ask, and the Law of Demeter

Instead of pulling data out of an object and deciding for it (`if (ticket.Status != Closed) {
ticket.Status = Resolved; ... }`), tell it what you want (`ticket.Resolve(now)`) and let it protect its
invariants. Code like `ticket.Team.Lead.Settings.Notifications.Email` reaches through four objects and
breaks when any of them changes shape; that's the Law of Demeter's "talk only to your immediate
friends". Both principles are about **putting behavior next to the data it guards**.

### Functional core, imperative shell

A pattern that subsumes several others: keep **decisions** in pure functions (no I/O, no clock, no
randomness) and push **effects** (database, HTTP, time, logging) to a thin outer layer that gathers
inputs, calls the core, and executes the result.

```csharp
// Pure core: easy to test with hundreds of cases, no mocks
public static class EscalationPolicy
{
    public static EscalationDecision Decide(TicketSummary t, DateTimeOffset now, SlaTarget target) =>
        (t.Status, now - t.CreatedAt) switch
        {
            (TicketStatus.Open, var age) when age > target.Response * 2 => EscalationDecision.PageLead,
            (TicketStatus.Open, var age) when age > target.Response     => EscalationDecision.Escalate,
            _                                                           => EscalationDecision.None,
        };
}

// Imperative shell: gathers data, applies the decision, performs effects
public sealed class SlaMonitorJob(BeaconDbContext db, INotifier notifier, TimeProvider clock)
{
    public async Task RunAsync(CancellationToken ct)
    {
        var now = clock.GetUtcNow();
        await foreach (var (ticket, target) in OpenTicketsWithTargets(db).WithCancellation(ct))
        {
            switch (EscalationPolicy.Decide(ticket, now, target))
            {
                case EscalationDecision.Escalate: await EscalateAsync(ticket.Id, ct); break;
                case EscalationDecision.PageLead: await notifier.PageLeadAsync(ticket.Id, ct); break;
            }
        }
    }
    // ...
}
```

The core is where the bugs that matter live, and it's now trivially testable. The shell is thin enough
that a few integration tests cover it.

---

## 5. The patterns that matter

The 1994 *Design Patterns* book catalogued 23 patterns, many of which compensate for missing language
features. C# now has lambdas, delegates, generics, pattern matching, records, `IEnumerable` and
`async`, so some patterns have become one-liners and others have disappeared into frameworks. The
useful question is not "which pattern is this?" but "**which force does this structure balance?**"

Here are the patterns a modern .NET full-stack engineer uses weekly, where you've already met them in
Beacon, and their modern form.

### Strategy: vary an algorithm

**Force:** one step of a process has several interchangeable implementations chosen at run time.

In Beacon: notification channels (`INotifier`), the AI model routing in Book XI (cheap model for
classification, stronger model for drafting), the SLA calendar (24×7 vs business hours per plan).

Modern form: often just a **delegate** (`Func<Ticket, TimeSpan>`) or a keyed DI service rather than a
class hierarchy.

```csharp
builder.Services.AddKeyedSingleton<IBusinessCalendar, AlwaysOnCalendar>("24x7");
builder.Services.AddKeyedSingleton<IBusinessCalendar, OfficeHoursCalendar>("business-hours");

public sealed class SlaClock([FromKeyedServices("business-hours")] IBusinessCalendar calendar) { /* ... */ }
```

### Decorator: add behavior around something without changing it

**Force:** cross-cutting concerns (caching, retries, logging, metrics, authorization) that should wrap
many operations without being written into each.

In Beacon: the `IChatClient` pipeline in Book XI is a decorator stack (`UseOpenTelemetry()`,
`UseDistributedCache()`, `UseFunctionInvocation()`), HTTP resilience handlers (`AddStandardResilienceHandler`)
are decorators around `HttpClient`, and ASP.NET Core middleware is a decorator chain over the request.

```csharp
public sealed class CachingArticleSearch(IArticleSearch inner, HybridCache cache) : IArticleSearch
{
    public ValueTask<IReadOnlyList<ArticleHit>> SearchAsync(string query, CancellationToken ct) =>
        cache.GetOrCreateAsync($"search:{query.Trim().ToLowerInvariant()}",
            async token => await inner.SearchAsync(query, token),
            cancellationToken: ct);
}
```

The caller depends on `IArticleSearch` and doesn't know caching exists. You can test the cache and the
search separately, and remove either without touching the other.

### Adapter and anti-corruption layer: translate between models

**Force:** an external system's model (a CRM, an identity provider, a legacy database, an AI vendor's
API) doesn't match yours, and you don't want its concepts leaking everywhere.

An adapter converts one interface into another. At larger scale (Chapter 2), an **anti-corruption
layer** is a module whose only job is translation, so that the CRM's "Case" with 140 fields becomes
Beacon's `Ticket` at one boundary and nowhere else. `Microsoft.Extensions.AI`'s `IChatClient` is an
adapter layer over many vendors' SDKs: application code speaks one model, adapters speak each vendor's.

### Factory: control creation

**Force:** creating an object requires decisions or dependencies the caller shouldn't know about.

Modern forms: static factory methods (`Ticket.Create(...)` returning `Result<Ticket>`, so invalid input
never produces an object), `IHttpClientFactory`, `IDbContextFactory<T>` for code that needs short-lived
contexts in singletons, and DI itself, which is a generalized factory. You rarely need an
`AbstractTicketFactoryFactory`.

### Observer and domain events: react without coupling

**Force:** when something happens, several other things should react, and the thing that happened
shouldn't know about them.

C# events (Book I, Chapter 6) are in-process observers. Beacon's domain events (`TicketAssigned`,
`TicketResolved`) are the same idea at the domain level, and the outbox (Book IV) turns them into
**durable** notifications that survive crashes. Chapter 4 takes this to message brokers. The cost
throughout is the same: control flow becomes implicit. "What happens when a ticket is resolved?" can no
longer be answered by reading `Resolve()`.

### Template method vs pipeline

Template method (a base class with an algorithm skeleton and abstract steps) is inheritance-based and
has largely given way to **pipelines** of composable steps: ASP.NET Core middleware, endpoint filters,
EF Core interceptors (Beacon's `DomainEventsToOutboxInterceptor`), and `IChatClient` builders. A
pipeline is a list of decorators you can reorder, test individually and configure per use.

### State machine

**Force:** an entity's allowed operations depend on its current state, and illegal transitions must be
impossible.

`Ticket` is a state machine (Open → InProgress → Resolved → Closed, with Reopen). For a handful of
states, methods with guard clauses are clearest. When transitions multiply, make the table explicit:

```csharp
private static readonly FrozenDictionary<(TicketStatus From, TicketAction Action), TicketStatus> Transitions =
    new Dictionary<(TicketStatus, TicketAction), TicketStatus>
    {
        [(TicketStatus.Open,       TicketAction.Assign)]  = TicketStatus.InProgress,
        [(TicketStatus.InProgress, TicketAction.Resolve)] = TicketStatus.Resolved,
        [(TicketStatus.Resolved,   TicketAction.Close)]   = TicketStatus.Closed,
        [(TicketStatus.Resolved,   TicketAction.Reopen)]  = TicketStatus.Open,
    }.ToFrozenDictionary();
```

A table is data: you can render it as a diagram, test it exhaustively, and show it to the support lead.

### Result vs exceptions

Not a GoF pattern, but a design decision you make constantly. Beacon uses `Result<T>` for **expected**
outcomes the caller must handle (not found, validation, conflict), and exceptions for **unexpected**
ones (database down, bugs, broken invariants via `DomainException`). Book I, Chapter 8 and Book XII,
Chapter 3 discuss the trade-off; the design principle is that **the signature should tell the truth**
about what can happen.

### Repository and Unit of Work

Covered in Book IV, Chapter 7. EF Core's `DbContext` *is* a unit of work, and `DbSet<T>` is close to a
repository. Beacon keeps a thin `ITicketRepository` because the domain owns it (DIP) and it loads the
`Ticket` aggregate as a whole. It does **not** wrap every table in a generic `IRepository<T>` with
`GetAll()`, `Find(Expression<...>)` and `Update()`; that hides EF Core's power behind a weaker API and
leaks `IQueryable` anyway.

### Mediator and CQRS-lite

Separating commands (change state, go through the domain model) from queries (read projections, go
straight to SQL or EF `Select`) is genuinely useful: reads and writes have different shapes and
different performance needs, and Beacon already does it informally (screen-oriented read endpoints in
Book VII, `Ticket` aggregate for writes).

The *mediator* library pattern (`mediator.Send(new CreateTicketCommand(...))`) is more debatable. It
gives you a uniform pipeline for cross-cutting behaviors. It also turns "go to definition" into a
search, and adds a layer between the endpoint and the code it calls.

> **🔄 Current (as of October 2026):** MediatR and AutoMapper, long the default .NET choices for mediator
> and object mapping, moved to commercial licenses for newer versions in 2025. Many teams responded by
> removing them: calling handlers directly from minimal API endpoints, using endpoint filters for
> cross-cutting concerns, and writing mapping code by hand or with source generators (Mapperly). Open
> source alternatives exist, including source-generated mediators. Check licensing before adopting any
> library at the core of your architecture.

Beacon's position: endpoints call application services directly. Cross-cutting concerns live in
middleware, endpoint filters and decorators. Mapping is explicit code, which is also where you notice
that a new field shouldn't be exposed.

---

## 6. In practice: refactoring a Beacon feature with these principles

Let's apply the chapter to a real change. The support lead asks: "When a ticket is resolved, the
customer should get a satisfaction survey, unless they're on the Free plan or it was resolved as
duplicate. And we're planning to add an SMS channel."

(Assume customers belong to organizations on Free, Team or Business plans; Beacon's billing module would
own that data.) The first, natural implementation adds code to the resolve endpoint:

```csharp
group.MapPost("/{id}/resolve", async (TicketId id, ResolveTicketRequest req, BeaconDbContext db,
    IEmailSender email, IHttpClientFactory http, ICurrentUser user, TimeProvider clock, CancellationToken ct) =>
{
    var ticket = await db.Tickets.Include(t => t.Comments).SingleOrDefaultAsync(t => t.Id == id, ct);
    if (ticket is null) return Results.NotFound();
    if (!user.IsAgent) return Results.Forbid();

    ticket.Resolve(clock.GetUtcNow());
    await db.SaveChangesAsync(ct);

    var reporter = await db.Users.FindAsync([ticket.ReporterId], ct);
    var org = await db.Organizations.FindAsync([reporter!.OrganizationId], ct);
    if (org!.Plan != "free" && req.Resolution != "duplicate")
    {
        await email.SendAsync(reporter.Email, "How did we do?", SurveyHtml(ticket), ct);
        // TODO: SMS
    }
    return Results.NoContent();
});
```

It works. Now list its problems as forces:

1. **Three reasons to change** (SRP): resolution workflow, survey eligibility rules, and delivery
   channels.
2. **Not atomic**: if the email send fails after `SaveChangesAsync`, the ticket is resolved but no survey
   is sent, and a retry by the client would hit "already resolved". If the send succeeds but the
   process crashes before responding, the agent retries and the customer gets two surveys.
3. **Latency**: the agent waits for an email provider.
4. **Hard to test**: the eligibility rule can only be tested through HTTP with a database and an email
   fake.
5. **SMS** will mean editing this endpoint again (OCP).

The refactored design uses principles from this chapter and patterns already in Beacon:

```csharp
// 1. Domain: Resolve() already raises TicketResolved (Book I). Add the resolution kind to it.
public sealed record TicketResolved(TicketId TicketId, UserId ReporterId, Resolution Resolution,
    DateTimeOffset At) : IDomainEvent;

// 2. Pure policy (functional core): one place, one reason to change, trivially testable
public static class SurveyPolicy
{
    public static bool ShouldSend(Resolution resolution, Plan plan, SurveyHistory history) =>
        plan != Plan.Free
        && resolution is not (Resolution.Duplicate or Resolution.Spam);
}

// 3. Handler for the event, run by the outbox worker (durable, retried, off the request path)
public sealed class SendSatisfactionSurvey(
    IReporterDirectory reporters,
    ISurveyRepository surveys,
    IEnumerable<ISurveyChannel> channels,        // Strategy + OCP: SMS is a new class
    TimeProvider clock) : IDomainEventHandler<TicketResolved>
{
    public async Task HandleAsync(TicketResolved e, CancellationToken ct)
    {
        if (await surveys.ExistsForAsync(e.TicketId, ct)) return;          // idempotent (Chapter 3)

        var reporter = await reporters.GetAsync(e.ReporterId, ct);
        if (!SurveyPolicy.ShouldSend(e.Resolution, reporter.Plan, reporter.SurveyHistory)) return;

        var survey = Survey.Create(e.TicketId, reporter.Id, clock.GetUtcNow());
        await surveys.AddAsync(survey, ct);

        foreach (var channel in channels.Where(c => c.Supports(reporter.Preferences)))
            await channel.SendAsync(survey, reporter, ct);
    }
}

// 4. The endpoint is back to one responsibility: HTTP in, use case, HTTP out
group.MapPost("/{id}/resolve", async (TicketId id, ResolveTicketRequest req,
    TicketService tickets, CancellationToken ct) =>
        (await tickets.ResolveAsync(id, req.Resolution, ct)).ToHttpResult())
    .RequireAuthorization(TicketOperations.Work);
```

What changed, and why each change pays rent:

| Change | Principle | Payoff |
|---|---|---|
| Survey sent from an outbox-driven handler | Observer/domain events, outbox | Atomic with the resolve; retried; off the request path |
| `SurveyPolicy` is a pure function | Functional core, SRP | Rule changes are one-line, tested exhaustively |
| `ISurveyChannel` collection | Strategy, OCP | SMS is a new class; nothing else changes |
| `ExistsForAsync` check | Idempotency (Chapter 3) | Outbox delivers at least once; no duplicate surveys |
| Thin endpoint + authorization policy | SRP, existing patterns | Same shape as every other endpoint |

And what we deliberately **didn't** do: no `ISurveyPolicy` interface (one implementation, pure
function), no generic `INotificationPipeline<T>`, no plugin system for channels, no mediator. Each
would add indirection without isolating a change we expect.

> **🔍 Investigation:** A week after shipping, someone notices customers with many tickets get a survey
> for each one. The fix is one line in `SurveyPolicy` (pass in a `SurveyHistory` and skip if one was sent
> in the last seven days) plus a few test cases. Compare that with finding and changing the rule inside
> the original endpoint, and testing it through HTTP. That's what "make the likely change cheap" means in
> practice. Notice too that the rule wasn't added *before* someone asked: speculative rules in policy
> code are worse than speculative abstractions, because they change behavior and nobody remembers why.

---

## 7. When patterns hurt

Patterns are solutions to recurring problems. Applied without the problem, they are pure cost. The
common failures:

**Pattern-first design.** Starting with "we'll use CQRS, MediatR, the specification pattern and
repositories" before knowing what the system does. The structure is decided by fashion rather than
forces, and the domain is squeezed to fit.

**Abstraction for a single implementation.** `ITicketService` with one implementation, registered in DI,
mocked in tests that then verify that the mock was called. The interface adds a file, an indirection
and a test that tests nothing. Exceptions: boundaries you need to fake (I/O, time, external services)
and public library APIs.

**Indirection that defeats navigation.** If "go to definition" lands on an interface and "find
implementations" lands on a generic handler resolved by reflection, the code is harder to read than a
direct call. Readability is a feature; static, navigable call graphs are part of it.

**The generic everything.** `IRepository<T>`, `BaseService<TEntity, TDto, TKey>`, `GenericController`.
Generic infrastructure assumes all entities behave alike. They don't: tickets have state machines,
articles have versions, users come from the identity provider. The generic base grows virtual hooks and
flags until it is the most complex class in the system.

**Layers that only forward.** Controller → Service → Manager → Repository → DAO where each method calls
the next with the same arguments. Every feature now touches five files. Layers should exist because
they contain different decisions.

**Patterns hiding a missing language feature you already have.** Visitor instead of pattern matching on
records, Command objects instead of delegates, Singleton classes instead of a DI singleton or a static
class, Builder for an object with three properties instead of an object initializer with `required`
members.

> **⚠️ What can go wrong:** The most expensive outcome of over-design is not the extra code. It's that
> the team stops changing the design because it's too complicated to understand, and starts working
> *around* it: special cases bolted onto the side, "temporary" bypasses of the layers. Over-designed
> systems decay into the same mess as under-designed ones, only with more files.

---

## 8. What can go wrong

- **Principles as rules.** "SOLID says every class needs an interface" — it doesn't. Principles are
  heuristics that need a force to act on. Ask "what change does this make cheap?" before applying one.
- **Premature generalization.** Abstracting after the first example, before you know what varies. You
  will guess wrong, and the wrong abstraction is harder to remove than duplication.
- **Shared kernels that grow.** A `Beacon.Common` project that everything references, and that slowly
  collects DTOs, helpers and constants for every feature. It becomes the most coupled module in the
  system. Keep shared code small, stable and genuinely generic (like `Result<T>` and `Error`).
- **Hidden temporal coupling.** Objects that must be initialized or called in a certain order, with
  nothing enforcing it. Prefer constructors that produce valid objects, and types that make the order
  explicit (a builder that returns a different type after a required step).
- **Anemic domain + fat services.** Data classes with public setters and 2,000-line services that
  enforce rules (sometimes) from the outside. Invariants end up duplicated across services, and one
  forgets. Book I, Chapter 3 covers the alternative.
- **Leaky abstractions ignored.** Every abstraction leaks: the repository hides SQL but not latency, the
  `IChatClient` hides the vendor but not token limits or rate limits. Design for the leak (timeouts,
  batching, budgets) rather than pretending it isn't there.
- **Framework-driven design.** Letting the shape of a framework (EF Core entities with public setters
  everywhere, React components that fetch and render and validate) define your domain. Frameworks are
  details; use them fully at the edges, keep them out of the core decisions.

---

## 9. When not to use it

> **🧭 When not to use it:** Skip most of this chapter's structure when the code is **small, short-lived
> or unlikely to change**: a migration script, a prototype to test an idea with users, a one-off data
> fix, a spike. Write the most direct code that works. Apply design when the code proves it will live:
> when it gets a second feature, a second developer, or a second production incident.

Also hold back on:

- **Stable, finished code.** If a module hasn't changed in two years and works, don't refactor it toward
  principles. Design effort belongs where change is happening.
- **Performance-critical inner loops.** Interfaces, virtual calls and allocations per call can matter in
  hot paths (Book I, Chapter 11). Concrete, sealed, sometimes duplicated code is the right trade-off
  there; measure first.
- **Tiny teams on small systems.** Many enterprise patterns assume multiple teams coordinating through
  contracts. Two developers in one repository can rely on the compiler and conversation.

---

## 10. How an experienced engineer thinks about this

- **Starts from forces, not patterns.** "What changes here, how often, and who asks for it?" The pattern
  name comes later, if at all.
- **Optimizes for the reader.** Code is a message to the next developer. Direct, navigable code with
  good names beats clever indirection.
- **Lets duplication sit until the shape is clear**, then extracts the abstraction from three real
  examples rather than one imagined future.
- **Pushes decisions into pure code** and effects to the edge, because pure code is where testing is
  cheap and bugs are visible.
- **Uses types as the first line of defense**: illegal states unrepresentable, signatures that tell the
  truth.
- **Treats design as continuous.** Refactoring toward a better structure happens in small steps,
  alongside features, guarded by tests. It's rarely a separate project.
- **Removes abstractions as readily as adding them.** Deleting an interface with one implementation, or
  inlining a "helper" with six flags, is good design work.
- **Knows the vocabulary for communication**, not compliance. Saying "this is a decorator" in a review is
  useful shorthand; insisting that code match a catalog entry is not.

---

## 11. Check yourself

**Questions**

1. Define coupling and cohesion. Why is concrete coupling inside one deployable often acceptable while
   coupling across a database shared by two teams is not?
2. What does the Single Responsibility Principle actually mean, and what's the most common misreading?
3. Why is the Open/Closed Principle only as good as your predictions? When should you apply it?
4. Give an example of a Liskov violation in .NET's base library and explain the fix.
5. What does "inversion" mean in the Dependency Inversion Principle? Who should own the interface?
6. Why can duplication be cheaper than abstraction? What's the warning sign that an abstraction is wrong?
7. What is "functional core, imperative shell", and how does it change how you test?
8. Name three patterns you've used in Beacon and the force each one balances.
9. List four ways patterns make code worse.

**Exercises**

1. Find a class in a codebase you know with more than one reason to change. Name the actors behind each
   reason and sketch the split. Then argue the opposite: is the split worth it?
2. Implement the survey refactoring from section 6 in Beacon, including tests for `SurveyPolicy`
   (table-driven with `[Theory]`) and an idempotency test for the handler.
3. Find an interface in your codebase with exactly one implementation and no test fake. Remove it. What
   got simpler? What, if anything, got harder?
4. Rewrite `Ticket`'s transitions as an explicit table, generate a Mermaid state diagram from it, and
   write a test that every `(status, action)` pair either transitions or returns a domain error.
5. Take a shared helper with three or more boolean parameters and inline it into its callers. Then
   look for the abstraction that actually emerges.

**Interview-style questions**

- "Explain SOLID. Which principle do you find most valuable, and which is most misused?"
- "When would you *not* introduce an interface?"
- "What's the difference between the Strategy and Decorator patterns? Give a real example of each."
- "Describe a time a design abstraction made a codebase worse. What did you do about it?"
- "How do you decide when to refactor duplicated code?"

---

## 12. Going deeper

- John Ousterhout, *A Philosophy of Software Design* (2nd ed.) — deep modules, information hiding, and a
  thoughtful counterpoint to "small classes everywhere".
- Andrew Hunt and David Thomas, *The Pragmatic Programmer* (20th anniversary ed.) — DRY, orthogonality,
  tracer bullets.
- Martin Fowler, *Refactoring* (2nd ed.) — the catalog of small, safe steps toward better design.
- Gamma, Helm, Johnson, Vlissides, *Design Patterns* — read for the "Motivation" and "Consequences"
  sections of each pattern, not the UML.
- Sandi Metz, ["The Wrong Abstraction"](https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction).
- Gary Bernhardt, "Boundaries" (talk) — the origin of "functional core, imperative shell".
- Scott Wlaschin, *Domain Modeling Made Functional* — making illegal states unrepresentable, in depth.

**Next:** [Application Architecture](02-application-architecture.md) applies these principles at the scale
of whole applications: layers, clean and hexagonal architecture, modular monoliths and microservices.
