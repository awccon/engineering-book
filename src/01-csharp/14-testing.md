# Testing

Most developers know *how* to write a test. The harder questions are what to test, at
which level, and how to write tests that help instead of hurt. Bad test suites are
common: slow, flaky, tightly coupled to implementation details, so every refactoring
breaks fifty tests that then get "fixed" by updating the expected values. Teams learn to
distrust them, and eventually stop running them.

A good test suite is different. It lets you change code confidently, documents how the
system behaves, catches regressions before users do, and runs fast enough that you run it
constantly. This chapter is about building that kind of suite, and it closes Book I by
testing everything Beacon has so far.

---

## 1. The problem: confidence to change code

Software changes constantly: new features, bug fixes, framework upgrades, refactoring.
Every change risks breaking something that used to work. Without tests, the only way to
know is manual checking (slow, incomplete) or waiting for users to report problems
(expensive, embarrassing).

Tests are **executable specifications** that check the system still does what it should.
Their value is measured in one currency: **how much confidence they give you, per unit of
cost** (time to write, time to run, time to maintain).

---

## 2. The mental model: kinds of tests

### By scope

| Kind | Tests | Speed | Confidence it gives | Typical tools |
|---|---|---|---|---|
| **Unit** | A class or function in isolation | Milliseconds | Logic is correct | xUnit, NUnit, MSTest |
| **Integration** | Several real components together (code + real database, HTTP pipeline) | 10s–100s of ms | Parts fit and infrastructure works | `WebApplicationFactory`, Testcontainers |
| **End-to-end** | The whole system through its real interface (browser, API) | Seconds | User journeys work | Playwright (Book VII) |

### The test pyramid (and its critics)

The classic **test pyramid** says: many unit tests, fewer integration tests, very few
end-to-end tests, because lower levels are faster and more precise.

```text
          ▲  E2E           few, slow, broad
         ▲▲▲ Integration   some
       ▲▲▲▲▲▲▲ Unit         many, fast, narrow
```

A popular alternative, the **testing trophy**, argues that in typical web applications
most bugs live at the *seams* (database queries, serialization, configuration, HTTP
binding), which unit tests with mocks can't catch. It recommends more integration tests.

Both are right about different code:

- **Complex logic** (rules, calculations, state machines, algorithms): unit tests. Fast,
  precise, many cases.
- **Glue code** (controllers, repositories, mapping, configuration): integration tests.
  Mocking a database to test a repository mostly tests your mock.
- **Critical user journeys** (sign in, create a ticket, pay): a few end-to-end tests.

> **🧱 Durable:** Test **behavior**, not **implementation**. A good test describes what
> the code does from the outside ("resolving a closed ticket fails") and survives any
> refactoring that preserves that behavior. A test that breaks whenever you rename a
> private method or reorder calls is testing implementation, and it's a liability.

---

## 3. Anatomy of a good unit test

### Arrange, Act, Assert

```csharp
[Fact]
public void Resolve_OpenTicket_SetsStatusToResolved()
{
    // Arrange
    var ticket = TicketBuilder.Open().Build();

    // Act
    ticket.Resolve(Now);

    // Assert
    Assert.Equal(TicketStatus.Resolved, ticket.Status);
}
```

### Properties of good tests (FIRST)

- **Fast**: milliseconds. Slow suites don't get run.
- **Isolated / Independent**: no shared state between tests; any order works; can run in
  parallel.
- **Repeatable**: same result every time, on every machine. No dependence on the clock,
  random numbers, network or test order.
- **Self-validating**: pass or fail automatically; no reading log output.
- **Timely**: written alongside the code, not months later.

### Naming

A test name should say what's being tested, under what conditions, with what expected
result. `MethodName_Condition_ExpectedResult` is one common convention; plain English
sentences work too (`Resolving_a_closed_ticket_is_rejected`). When a test fails in CI, the
name alone should tell you what broke.

### One behavior per test

Several asserts are fine if they check *one behavior* (status changed *and* an event was
raised). Testing three unrelated behaviors in one test hides which one failed.

---

## 4. xUnit essentials

xUnit is the most widely used test framework in modern .NET (and what the .NET team uses).

```bash
dotnet new xunit -n Beacon.Core.Tests -o tests/Beacon.Core.Tests
dotnet sln add tests/Beacon.Core.Tests
dotnet add tests/Beacon.Core.Tests reference src/Beacon.Core
dotnet test
```

> **🔄 Current (as of October 2026):** xUnit v3 is the current major version (packages
> `xunit.v3`), with tests running as standalone executables and support for Microsoft
> Testing Platform. The `dotnet new xunit` template may still target v2 depending on your
> SDK; the concepts and attributes below are the same in both.

Key features:

```csharp
public sealed class SlaRulesTests
{
    [Fact]                                       // a single test
    public void Urgent_target_is_one_hour()
        => Assert.Equal(TimeSpan.FromHours(1), SlaRules.ResponseTarget(TicketPriority.Urgent));

    [Theory]                                     // a parameterized test
    [InlineData(TicketPriority.Urgent, 1)]
    [InlineData(TicketPriority.High, 4)]
    [InlineData(TicketPriority.Normal, 24)]
    [InlineData(TicketPriority.Low, 72)]
    public void Response_targets_match_policy(TicketPriority priority, int hours)
        => Assert.Equal(TimeSpan.FromHours(hours), SlaRules.ResponseTarget(priority));
}
```

- xUnit creates a **new instance of the test class for every test**, so instance fields
  are fresh each time. Setup goes in the constructor; cleanup in `Dispose` (or
  `IAsyncLifetime` for async setup and teardown).
- **Shared expensive setup** (a database container) uses *fixtures*:
  `IClassFixture<T>` for one class, collection fixtures for several.
- Tests in different classes run **in parallel** by default; tests in the same class run
  sequentially.

Many teams add **FluentAssertions** or **Shouldly** for more readable assertions
(`ticket.Status.Should().Be(TicketStatus.Resolved)`). Plain `Assert` is perfectly fine.

> **🔄 Current (as of October 2026):** FluentAssertions changed to a commercial license
> for version 8 and later. Check licensing before adopting it; Shouldly and plain
> `Assert` are free alternatives.

---

## 5. Test doubles

When the code under test depends on something slow, non-deterministic or external, you
replace it in tests with a **test double**:

| Kind | What it does | Example |
|---|---|---|
| **Stub** | Returns canned answers | A repository that always returns the same ticket |
| **Fake** | A working, simplified implementation | An in-memory repository; `FakeTimeProvider` |
| **Spy** | Records how it was called | A notifier that stores sent messages |
| **Mock** | Pre-programmed with expectations; the test verifies calls | Moq/NSubstitute: "verify `NotifyAsync` was called once with 'maria'" |

### Prefer fakes and state verification

Compare two ways of testing that assigning a ticket notifies the assignee:

```csharp
// Mock-based (interaction testing)
var notifier = Substitute.For<INotifier>();
await service.AssignAsync(id, "maria", ct);
await notifier.Received(1).NotifyAsync("maria", Arg.Any<string>(), Arg.Any<CancellationToken>());

// Fake-based (state testing)
var notifier = new RecordingNotifier();
await service.AssignAsync(id, "maria", ct);
Assert.Contains(notifier.Sent, m => m.Recipient == "maria");
```

Both work. Fakes are reusable across many tests, read naturally and don't couple tests to
exact method signatures. Mocking libraries (NSubstitute, Moq) are handy for one-off stubs
and for verifying interactions that *are* the behavior (an email must be sent exactly
once).

> **🧭 When not to mock:** Don't mock what you don't own (`DbContext`, `HttpClient`,
> framework types). Mocks of complex third-party APIs encode your *assumptions* about
> them, and the test passes even when the assumption is wrong. Test those seams with
> integration tests against the real thing (a real PostgreSQL in a container, a real
> HTTP pipeline), and keep fakes for your own interfaces.

### Control time and randomness

Code that calls `DateTime.UtcNow` or `Random.Shared` directly can't be tested
deterministically. Inject `TimeProvider` (as Beacon does) and use `FakeTimeProvider` in
tests:

```csharp
var clock = new FakeTimeProvider(new DateTimeOffset(2026, 10, 1, 9, 0, 0, TimeSpan.Zero));
clock.Advance(TimeSpan.FromHours(5));   // time travel, deterministically
```

`FakeTimeProvider` also controls timers and `Task.Delay` calls that use the provider,
so you can test timeouts without waiting.

---

## 6. Integration tests

Integration tests exercise real infrastructure. You'll write many in later books; here's
the shape.

- **ASP.NET Core**: `WebApplicationFactory<Program>` starts your whole application
  in memory and gives you an `HttpClient` to call it. You can replace specific services
  for tests. (Book III.)
- **Databases**: **Testcontainers** starts a real PostgreSQL in Docker for the test run.
  Tests run real SQL, real migrations, real constraints. (Book IV.)

Integration tests need more care about **isolation**: each test should create its own
data (unique IDs) or reset the database, or tests will interfere with each other when run
in parallel.

---

## 7. What to test, and what not to

**Test thoroughly:**

- Business rules and invariants (every branch of `Ticket`'s state transitions).
- Calculations and edge cases (SLA boundaries, empty inputs, maximum sizes).
- Bugs you've fixed: a **regression test** first, reproducing the bug, then the fix.
- Error paths (not found, validation, conflict).

**Test lightly or not at all:**

- Trivial code (auto-properties, simple DTOs, one-line delegations).
- Framework behavior (that ASP.NET Core routes requests; Microsoft tests that).
- Private methods directly. Test them through the public behavior that uses them. If a
  private method is complex enough to want its own tests, it probably wants to be its own
  class.

### Coverage

Code coverage (`dotnet test --collect:"XPlat Code Coverage"`) tells you which lines ran
during tests. It's useful for finding **untested areas**. It's a poor **target**: 100%
coverage with weak assertions proves only that code executed, not that it's correct. Use
coverage to ask questions ("why is the refund logic at 20%?"), not as a goal.

**Mutation testing** (Stryker.NET) is a stronger measure: it changes your code (flips a
`>` to `>=`, removes a line) and checks whether any test fails. Surviving mutants reveal
tests that don't really verify behavior.

---

## 8. Test-driven development (TDD)

TDD is a workflow: write a failing test, write the simplest code that passes it, then
refactor. *Red, green, refactor.*

Benefits: you only write code you need; designs end up testable by construction; you get
fast feedback. It works especially well for logic with clear inputs and outputs (parsers,
rules, calculations) and for bug fixes (write the failing regression test first).

It's a tool, not a religion. Exploratory work, UI layout and spikes are often better done
first and tested after. Many experienced developers use TDD selectively: always for bugs
and complex logic, less for glue code.

---

## 9. In practice: Beacon's test suite

Time to test what Book I built. Create the project as in section 4, and add the
`Microsoft.Extensions.TimeProvider.Testing` package for `FakeTimeProvider`.

### A test data builder

Tests need tickets in many states. A **builder** keeps tests short and focused on what
matters for each test:

```csharp
// tests/Beacon.Core.Tests/TicketBuilder.cs
using Beacon.Core.Tickets;

namespace Beacon.Core.Tests;

internal sealed class TicketBuilder
{
    public static readonly DateTimeOffset DefaultCreated = new(2026, 10, 1, 9, 0, 0, TimeSpan.Zero);

    private int _id = 1;
    private string _title = "Cannot log in";
    private TicketPriority _priority = TicketPriority.Normal;
    private DateTimeOffset _created = DefaultCreated;
    private string? _assignee;
    private TicketStatus _status = TicketStatus.Open;

    public static TicketBuilder Open() => new();

    public TicketBuilder WithId(int id) { _id = id; return this; }
    public TicketBuilder WithPriority(TicketPriority p) { _priority = p; return this; }
    public TicketBuilder CreatedAt(DateTimeOffset at) { _created = at; return this; }
    public TicketBuilder AssignedTo(string who) { _assignee = who; return this; }
    public TicketBuilder Resolved() { _status = TicketStatus.Resolved; return this; }
    public TicketBuilder Closed() { _status = TicketStatus.Closed; return this; }

    public Ticket Build()
    {
        var t = new Ticket(new TicketId(_id), _title, _priority, _created);
        if (_assignee is not null) t.AssignTo(_assignee, _created);
        if (_status is TicketStatus.Resolved or TicketStatus.Closed) t.Resolve(_created);
        if (_status is TicketStatus.Closed) t.Close();
        t.ClearDomainEvents();
        return t;
    }
}
```

Note that the builder uses the **public API** to reach each state. It doesn't use
reflection to poke private fields. If the rules change, the builder fails loudly instead
of producing impossible tickets.

### Domain tests

```csharp
// tests/Beacon.Core.Tests/Tickets/TicketTests.cs
using Beacon.Core.Common;
using Beacon.Core.Tickets;

namespace Beacon.Core.Tests.Tickets;

public sealed class TicketTests
{
    private static readonly DateTimeOffset Now = TicketBuilder.DefaultCreated.AddHours(1);

    [Theory]
    [InlineData("")]
    [InlineData("   ")]
    public void Title_is_required(string title)
        => Assert.ThrowsAny<ArgumentException>(() =>
            new Ticket(new TicketId(1), title, TicketPriority.Normal, Now));

    [Fact]
    public void New_ticket_is_open_and_unassigned()
    {
        var t = TicketBuilder.Open().Build();
        Assert.Equal(TicketStatus.Open, t.Status);
        Assert.Null(t.Assignee);
        Assert.True(t.IsActive);
    }

    [Fact]
    public void Assigning_an_open_ticket_moves_it_to_in_progress_and_raises_event()
    {
        var t = TicketBuilder.Open().Build();

        t.AssignTo("maria", Now);

        Assert.Equal(TicketStatus.InProgress, t.Status);
        Assert.Equal("maria", t.Assignee);
        var e = Assert.IsType<TicketAssigned>(Assert.Single(t.DomainEvents));
        Assert.Equal("maria", e.Assignee);
    }

    [Fact]
    public void Closed_ticket_rejects_comments()
    {
        var t = TicketBuilder.Open().Closed().Build();
        Assert.Throws<DomainException>(() => t.AddComment(new Comment("omar", "Still broken", Now)));
    }

    [Fact]
    public void Reopening_a_resolved_assigned_ticket_returns_it_to_in_progress()
    {
        var t = TicketBuilder.Open().AssignedTo("maria").Resolved().Build();
        t.Reopen();
        Assert.Equal(TicketStatus.InProgress, t.Status);
    }
}
```

### SLA boundary tests

Boundaries are where bugs live. Is a ticket breaching at *exactly* its target time?

```csharp
// tests/Beacon.Core.Tests/Tickets/SlaRulesTests.cs
public sealed class SlaRulesTests
{
    private static readonly DateTimeOffset Created = TicketBuilder.DefaultCreated;

    [Theory]
    [InlineData(59, false)]
    [InlineData(60, false)]     // exactly at the target: not yet breaching
    [InlineData(61, true)]
    public void Urgent_ticket_breaches_after_one_hour(int minutes, bool expected)
    {
        var t = TicketBuilder.Open().WithPriority(TicketPriority.Urgent).Build();
        Assert.Equal(expected, SlaRules.IsBreaching(t, Created.AddMinutes(minutes)));
    }

    [Fact]
    public void Resolved_tickets_never_breach()
    {
        var t = TicketBuilder.Open().WithPriority(TicketPriority.Urgent).Resolved().Build();
        Assert.False(SlaRules.IsBreaching(t, Created.AddDays(30)));
    }
}
```

Writing the boundary test forces a decision ("is exactly 60 minutes a breach?") that the
code made implicitly with `>`. Tests make such decisions visible.

### Application service tests with fakes

```csharp
// tests/Beacon.Core.Tests/Fakes.cs
using Beacon.Core.Tickets;

namespace Beacon.Core.Tests;

internal sealed class FakeTicketRepository : ITicketRepository
{
    private readonly Dictionary<TicketId, Ticket> _store = new();
    public int SaveCount { get; private set; }

    public FakeTicketRepository(params Ticket[] tickets)
    {
        foreach (var t in tickets) _store[t.Id] = t;
    }

    public Task<Ticket?> FindAsync(TicketId id, CancellationToken ct = default)
        => Task.FromResult(_store.GetValueOrDefault(id));

    public Task SaveAsync(Ticket ticket, CancellationToken ct = default)
    {
        _store[ticket.Id] = ticket;
        SaveCount++;
        return Task.CompletedTask;
    }
}
```

```csharp
// tests/Beacon.Core.Tests/Tickets/TicketServiceTests.cs
using Beacon.Core.Tickets;
using Microsoft.Extensions.Time.Testing;

namespace Beacon.Core.Tests.Tickets;

public sealed class TicketServiceTests
{
    private readonly FakeTimeProvider _clock = new(TicketBuilder.DefaultCreated.AddHours(2));

    [Fact]
    public async Task Adding_a_comment_saves_the_ticket()
    {
        var ticket = TicketBuilder.Open().Build();
        var repo = new FakeTicketRepository(ticket);
        var service = new TicketService(repo, _clock);

        var result = await service.AddCommentAsync(ticket.Id, "maria", "  Looking into it  ");

        Assert.True(result.IsSuccess);
        var comment = Assert.Single(result.Value.Comments);
        Assert.Equal("Looking into it", comment.Body);          // trimmed
        Assert.Equal(_clock.GetUtcNow(), comment.CreatedAt);    // uses the injected clock
        Assert.Equal(1, repo.SaveCount);
    }

    [Fact]
    public async Task Missing_ticket_returns_not_found_without_saving()
    {
        var repo = new FakeTicketRepository();
        var service = new TicketService(repo, _clock);

        var result = await service.AddCommentAsync(new TicketId(404), "maria", "Hello?");

        Assert.False(result.IsSuccess);
        Assert.Equal("not_found", result.Error!.Code);
        Assert.Equal(0, repo.SaveCount);
    }

    [Fact]
    public async Task Closed_ticket_returns_conflict_instead_of_throwing()
    {
        var ticket = TicketBuilder.Open().Closed().Build();
        var service = new TicketService(new FakeTicketRepository(ticket), _clock);

        var result = await service.AddCommentAsync(ticket.Id, "maria", "Reopen please");

        Assert.Equal("conflict", result.Error!.Code);
    }
}
```

These tests verify the policy from Chapter 8: expected failures come back as results,
with no save, and nothing throws.

### An architecture test

One more kind of test is worth knowing. Beacon's core must never depend on
infrastructure. A test can enforce that with reflection (Chapter 12):

```csharp
// tests/Beacon.Core.Tests/ArchitectureTests.cs
public sealed class ArchitectureTests
{
    [Fact]
    public void Core_does_not_reference_infrastructure()
    {
        var referenced = typeof(Beacon.Core.Tickets.Ticket).Assembly
            .GetReferencedAssemblies()
            .Select(a => a.Name!)
            .ToList();

        Assert.DoesNotContain(referenced, n => n.StartsWith("Microsoft.EntityFrameworkCore"));
        Assert.DoesNotContain(referenced, n => n.StartsWith("Microsoft.AspNetCore"));
        Assert.DoesNotContain(referenced, n => n.StartsWith("Npgsql"));
    }
}
```

When someone adds `using Microsoft.EntityFrameworkCore;` to a domain class in Book IV,
this test fails, and the architecture is protected by CI rather than by memory.
(Libraries such as NetArchTest and ArchUnitNET make richer rules easy.)

Run everything:

```bash
dotnet test
```

---

## 10. What can go wrong

- **Testing implementation details**, so refactoring breaks tests that should have passed.
- **Over-mocking**: tests that verify a mock was called in a particular way, proving
  nothing about real behavior.
- **Flaky tests** from time, randomness, shared state, ordering or real network calls.
  A flaky test is worse than none: it trains everyone to ignore failures. Fix or delete.
- **Slow suites** that developers stop running locally.
- **Tests without meaningful assertions**, written to hit a coverage number.
- **Giant test setup** that obscures what a test is about. Use builders.
- **Testing only the happy path.** Bugs live in edge cases and error paths.

---

## 11. How an experienced engineer thinks about this

- **Tests are for confidence to change.** If a test doesn't increase confidence or
  actively makes change harder, it's a cost.
- **Match test type to code type**: unit tests for logic, integration tests for seams,
  a few E2E tests for critical journeys.
- **Design for testability.** Inject time, I/O and external services; keep logic in pure
  functions and domain objects. Code that's hard to test is usually telling you something
  about its design.
- **Every bug gets a regression test.**
- **Treat test code as production code**: readable, maintained, refactored.

---

## 12. Check yourself

**Questions**

1. What's the difference between unit, integration and end-to-end tests? Where does each
   give the most value?
2. Explain the test pyramid and the testing trophy. When is each perspective right?
3. What's the difference between a stub, a fake, a spy and a mock?
4. Why shouldn't you mock types you don't own, such as `DbContext`?
5. Why is 100% code coverage a poor goal?
6. How do you make code that depends on the current time testable?

**Exercises**

1. Add tests for `TriageQueue` (Chapter 5): urgent before normal, older before newer for
   the same priority, and resolved tickets skipped.
2. Add tests for `AgentWorkloadReport` (Chapter 7), including case-insensitive grouping.
3. Write a test for `Ticket.Escalate` from Chapter 3's exercise *before* implementing
   it (TDD).
4. Run Stryker.NET (`dotnet tool install -g dotnet-stryker`) on `Beacon.Core` and look
   at the surviving mutants. Add tests to kill them.

**Interview-style questions**

- "How do you decide what to unit test and what to integration test?"
- "What makes a test flaky? How do you fix flaky tests?"
- "What's TDD? Do you use it?"
- "How would you test code that sends emails?"

---

## 13. Going deeper

- [Microsoft docs: Unit testing best practices for .NET](https://learn.microsoft.com/dotnet/core/testing/unit-testing-best-practices)
- [xUnit documentation](https://xunit.net/)
- Vladimir Khorikov, *Unit Testing: Principles, Practices, and Patterns* — the best book on
  writing valuable tests, with .NET examples.
- Kent Beck, *Test-Driven Development: By Example*.

---

## Book I wrap-up

You now have a mental model of the platform from the bottom up: how C# becomes machine
code, how values and objects live in memory, how to design types and behavior, how
generics, collections, delegates and LINQ work, how to handle failure, how async and
concurrency really work, what the GC costs, how frameworks inspect code, how objects get
wired together, and how to test all of it.

Beacon has a domain model (`Ticket` with rules and events), application services with
result types, a triage queue, reports, an audit log, an indexing pipeline, a composition
root, and a test suite.

**Next:** [Book II — Git & Developer Workflow](../02-git/README.md) puts Beacon under
version control and introduces the collaboration workflow you'll use for the rest of the
book.
