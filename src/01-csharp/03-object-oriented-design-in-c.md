# Object-Oriented Design in C#

Almost every C# developer can define a class, implement an interface and override a
method. Far fewer can explain *why* a design uses an interface here and inheritance
there, or recognize when an object-oriented design is making the code worse.

This chapter is about the judgment behind the syntax: what problems OOP is good at
solving, what encapsulation actually protects, why "favor composition over inheritance"
became the standard advice, and when plain functions and data beat objects.

---

## 1. The problem OOP solves

As programs grow, two things become hard:

1. **Keeping data valid.** If any code anywhere can change a ticket's status, then the
   rule "a closed ticket can't be resolved" has to be enforced everywhere, and it won't be.
2. **Changing one part without breaking others.** If code depends on *how* something is
   done rather than *what* it does, every change ripples outward.

Object-oriented design attacks both by bundling **data with the operations that keep it
valid**, and by letting code depend on **contracts** (what an object promises) instead of
**implementations** (how it fulfills the promise).

The four classic pillars map onto those goals:

| Pillar | Goal it serves |
|---|---|
| **Encapsulation** | Keep data valid by controlling how it changes |
| **Abstraction** | Depend on *what*, not *how* |
| **Polymorphism** | Let different implementations satisfy one contract |
| **Inheritance** | Reuse and specialize behavior (useful, but the most overused) |

---

## 2. Encapsulation: protecting invariants

An **invariant** is a rule that must always be true for an object to be valid:

- A ticket's title is never empty.
- A closed ticket is never reopened as "resolved".
- An order's total equals the sum of its lines.

Encapsulation means the object is the *only* thing that can break its invariants, so
it's the only place you need to enforce them.

Compare two designs:

```csharp
// Anemic: data bag, rules live elsewhere (or nowhere)
public class Ticket
{
    public string Title { get; set; } = "";
    public TicketStatus Status { get; set; }
}

// Somewhere in a controller...
ticket.Status = TicketStatus.Resolved;   // was it closed? who checks?
```

```csharp
// Encapsulated: the type guards its own rules
public sealed class Ticket
{
    public TicketStatus Status { get; private set; }

    public void Resolve()
    {
        if (Status == TicketStatus.Closed)
            throw new InvalidOperationException("A closed ticket can't be resolved.");
        Status = TicketStatus.Resolved;
    }
}
```

In the second design, there is exactly one way to resolve a ticket, and it's always
correct. You can search the codebase for `Resolve(` and find every place it happens.

### Tell, don't ask

A useful heuristic: instead of *asking* an object for its data and making decisions
outside it, *tell* it what you want and let it decide.

```csharp
// Ask (logic leaks out of the object)
if (ticket.Status == TicketStatus.Open && ticket.Assignee is null)
    ticket.Status = TicketStatus.InProgress;

// Tell (logic stays with the data)
ticket.AssignTo("maria");
```

### Encapsulating collections

A common leak: exposing a mutable list.

```csharp
public List<Comment> Comments { get; } = new();   // anyone can Add, Remove, Clear
```

Expose a read-only view and provide methods for changes:

```csharp
private readonly List<Comment> _comments = new();
public IReadOnlyList<Comment> Comments => _comments;

public void AddComment(Comment comment)
{
    if (Status == TicketStatus.Closed)
        throw new InvalidOperationException("Closed tickets can't receive comments.");
    _comments.Add(comment);
}
```

> **🧱 Durable:** Encapsulation is not about getters and setters. A class with private
> fields and public getters/setters for each of them is no more encapsulated than one
> with public fields. Encapsulation is about **which operations exist**: the public
> surface should be the set of meaningful things you can do, each of which leaves the
> object valid.

---

## 3. Abstraction and interfaces

An **interface** is a contract: a set of members a type promises to provide, with no
implementation details.

```csharp
public interface ITicketRepository
{
    Task<Ticket?> FindAsync(TicketId id, CancellationToken ct = default);
    Task AddAsync(Ticket ticket, CancellationToken ct = default);
}
```

Code that depends on `ITicketRepository` doesn't know or care whether tickets live in
PostgreSQL, in memory or behind an HTTP API. That decoupling buys three things:

1. **Substitutability**: swap implementations (in-memory for tests, PostgreSQL for
   production).
2. **Parallel work**: one person builds the repository while another builds the code
   that uses it.
3. **Stable boundaries**: changes inside an implementation don't ripple outward.

### Modern interface features

Interfaces have grown since C# 8:

- **Default interface methods**: an interface can provide a default implementation,
  which lets library authors add members without breaking implementers.
- **Static abstract members** (C# 11): interfaces can require static members, which is
  what powers *generic math* (`INumber<T>`; see Chapter 4).

Use both sparingly. If an interface accumulates lots of default behavior, you probably
want an abstract base class or a separate helper.

### Interface segregation

Small, focused interfaces are easier to implement, mock and understand than large ones:

```csharp
// Too broad: a read-only report needs to implement Delete?
public interface ITicketStore { /* Find, List, Add, Update, Delete, Export, Archive... */ }

// Focused
public interface ITicketReader { Task<Ticket?> FindAsync(TicketId id, CancellationToken ct); }
public interface ITicketWriter { Task AddAsync(Ticket t, CancellationToken ct); }
```

### When an interface is the wrong tool

The most common over-engineering in C# codebases is an interface for every class:
`IUserService` with exactly one implementation, `UserService`, that will never have
another.

> **🧭 When not to use it:** Add an interface when you have (or clearly will have) more
> than one implementation, when you need a seam for testing something slow or external
> (database, HTTP, clock, file system), or when it marks a boundary between modules.
> Don't add one "just in case." Extracting an interface later takes minutes with any IDE;
> maintaining hundreds of single-implementation interfaces costs time forever.

---

## 4. Polymorphism

**Polymorphism** means one call site works with many types. C# offers several kinds:

| Kind | Mechanism | Example |
|---|---|---|
| Subtype polymorphism | `virtual` / `override`, interfaces | `INotifier.SendAsync` → email or Slack |
| Parametric polymorphism | Generics | `List<T>` works with any `T` (Chapter 4) |
| Ad-hoc polymorphism | Overloading | `Console.WriteLine(int)` vs `(string)` |
| Pattern matching | `switch` on types/shapes | Handling a closed set of cases |

Subtype polymorphism replaces conditionals with dispatch:

```csharp
// Before: every new channel means editing this switch
switch (channel)
{
    case "email": SendEmail(msg); break;
    case "slack": PostToSlack(msg); break;
}

// After: each channel is a type; adding one doesn't touch existing code
public interface INotifier { Task SendAsync(Notification n, CancellationToken ct); }
public sealed class EmailNotifier : INotifier { /* ... */ }
public sealed class SlackNotifier : INotifier { /* ... */ }
```

### The expression problem: when switches are better

Polymorphism makes it easy to **add new types** (a new notifier) but hard to **add new
operations** (every implementation needs the new method). Pattern matching is the
opposite: easy to add operations, hard to add types.

When the set of cases is **closed and stable** (ticket statuses, the result of a
validation, the kinds of event a system emits) and you'll write *many operations* over
them, a `switch` expression is clearer than spreading logic across classes:

```csharp
public static string Badge(TicketStatus status) => status switch
{
    TicketStatus.Open       => "🟢 Open",
    TicketStatus.InProgress => "🟡 In progress",
    TicketStatus.Resolved   => "🔵 Resolved",
    TicketStatus.Closed     => "⚪ Closed",
    _ => throw new ArgumentOutOfRangeException(nameof(status)),
};
```

> **🧱 Durable:** Choose polymorphism when the set of *types* will grow. Choose pattern
> matching when the set of *operations* will grow. This trade-off exists in every
> language and is worth recognizing explicitly.

---

## 5. Inheritance and why composition usually wins

Inheritance lets a class reuse and specialize another class's behavior:

```csharp
public abstract class Notification
{
    public required string Recipient { get; init; }
    public abstract string Render();
}

public sealed class TicketAssignedNotification : Notification
{
    public required TicketId TicketId { get; init; }
    public override string Render() => $"Ticket {TicketId} was assigned to you.";
}
```

That's a reasonable use: a shallow hierarchy, an abstract base defining a contract, and
leaf classes that are `sealed`.

The trouble starts with deeper or "convenience" hierarchies:

```text
BaseController
 └── AuthenticatedController
      └── CrudController<T>
           └── TicketController
                └── AdminTicketController
```

Problems you'll see in such hierarchies:

- **Fragile base class.** A change to a base class can break subclasses you've never seen.
  Subclasses depend on the base's *implementation details* (which methods call which,
  in what order), not just its contract.
- **Forced coupling.** `AdminTicketController` inherits everything from four ancestors,
  including behavior it doesn't need.
- **Single inheritance.** A class can only have one base, so you can't combine
  "auditable" and "cacheable" behavior from two bases.
- **Reading is hard.** Understanding one method means reading five files.

**Composition** builds behavior by *holding* other objects rather than *being* them:

```csharp
public sealed class TicketService(
    ITicketRepository repository,
    INotifier notifier,
    TimeProvider clock)
{
    public async Task AssignAsync(TicketId id, string assignee, CancellationToken ct)
    {
        var ticket = await repository.FindAsync(id, ct)
            ?? throw new KeyNotFoundException($"Ticket {id} not found.");
        ticket.AssignTo(assignee);
        await notifier.SendAsync(new TicketAssigned(id, assignee, clock.GetUtcNow()), ct);
    }
}
```

`TicketService` gets behavior from three collaborators, each replaceable, each testable,
none of which it inherits from. (The `TicketService(...)` syntax is a **primary
constructor**, available on classes since C# 12; the parameters are captured and usable
throughout the class.)

> **🧭 When not to use inheritance:** Avoid inheritance for code reuse alone. Use it when
> there's a genuine *is-a* relationship, the base class is designed for extension
> (abstract members, documented virtual methods), and the hierarchy stays shallow
> (one or two levels). Mark classes `sealed` by default; unsealing later is easy,
> sealing later is a breaking change.

### Liskov substitution

The rule that makes inheritance safe: **a subclass must be usable anywhere its base
class is, without surprises.** The classic violation:

```csharp
public class Rectangle { public virtual int Width { get; set; } public virtual int Height { get; set; } }
public class Square : Rectangle { /* setting Width also sets Height */ }

void Stretch(Rectangle r) { r.Width = 10; r.Height = 5; Debug.Assert(r.Width * r.Height == 50); } // fails for Square
```

Mathematically a square *is* a rectangle; behaviorally, a mutable `Square` isn't a
substitutable `Rectangle`. Inheritance models **behavior**, not taxonomy.

---

## 6. Rich domain models vs anemic models

There are two broad styles in business applications:

- **Anemic model**: entities are data bags; logic lives in "services."
- **Rich domain model**: entities enforce their own rules; services coordinate.

```text
 Anemic                              Rich
 ┌──────────────┐                    ┌───────────────────────┐
 │ TicketService│  sets fields on    │ TicketService         │ coordinates:
 │  - Resolve() │ ─────────────────► │  load, call, save,    │ ──► Ticket.Resolve()
 │  - Assign()  │    Ticket {get;set}│  notify               │     enforces the rules
 └──────────────┘                    └───────────────────────┘
```

The rich style keeps rules next to the data, so they can't be bypassed. The anemic style
is simpler for CRUD-heavy apps with little behavior.

> **🧭 When not to use a rich model:** If a feature is genuinely just "store this form,
> show it later," wrapping it in a rich domain model adds ceremony without protecting
> anything. Use rich models where there are real invariants and workflows (Beacon's
> tickets: status transitions, assignment, SLAs) and keep simple data simple.

---

## 7. When OOP is the wrong tool

C# is multi-paradigm. Some problems are clearer as **data plus functions**:

- **Transformations and pipelines** (parse → validate → map → serialize) read better as
  a chain of functions over immutable data, which is what LINQ is (Chapter 7).
- **Calculations** with no state (pricing rules, tax, formatting) can be `static`
  methods on a `static class`. Not everything needs to be an injectable service.
- **Closed sets of cases** often fit records plus pattern matching better than a class
  hierarchy (section 4).

```csharp
public static class SlaRules
{
    public static TimeSpan ResponseTarget(TicketPriority priority) => priority switch
    {
        TicketPriority.Urgent => TimeSpan.FromHours(1),
        TicketPriority.High   => TimeSpan.FromHours(4),
        TicketPriority.Normal => TimeSpan.FromDays(1),
        TicketPriority.Low    => TimeSpan.FromDays(3),
        _ => throw new ArgumentOutOfRangeException(nameof(priority)),
    };
}
```

A pure function like this is trivially testable, thread-safe and obvious. Wrapping it in
`ISlaRuleProvider` with one implementation would add nothing.

---

## 8. In practice: Beacon's domain

Let's add the next pieces of Beacon's domain, applying the chapter:

- A ticket has a **priority** and a list of **comments**.
- Closed tickets can't be commented on or reassigned.
- Status transitions follow rules (you can't go from `Closed` back to `InProgress`).
- When a ticket is assigned, the assignee should be notified, which is a capability
  supplied from outside the domain.

```csharp
// src/Beacon.Core/Tickets/TicketPriority.cs
namespace Beacon.Core.Tickets;

public enum TicketPriority { Low, Normal, High, Urgent }
```

```csharp
// src/Beacon.Core/Tickets/Comment.cs
namespace Beacon.Core.Tickets;

public sealed record Comment(string Author, string Body, DateTimeOffset CreatedAt);
```

```csharp
// src/Beacon.Core/Tickets/Ticket.cs
namespace Beacon.Core.Tickets;

public sealed class Ticket
{
    private readonly List<Comment> _comments = [];

    public Ticket(TicketId id, string title, TicketPriority priority, DateTimeOffset createdAt)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(title);
        Id = id;
        Title = title.Trim();
        Priority = priority;
        CreatedAt = createdAt;
        Status = TicketStatus.Open;
    }

    public TicketId Id { get; }
    public string Title { get; private set; }
    public TicketPriority Priority { get; private set; }
    public TicketStatus Status { get; private set; }
    public string? Assignee { get; private set; }
    public DateTimeOffset CreatedAt { get; }
    public IReadOnlyList<Comment> Comments => _comments;

    public bool IsActive => Status is TicketStatus.Open or TicketStatus.InProgress;

    public void AssignTo(string assignee)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(assignee);
        EnsureNotClosed();
        Assignee = assignee;
        if (Status == TicketStatus.Open) Status = TicketStatus.InProgress;
    }

    public void AddComment(Comment comment)
    {
        ArgumentNullException.ThrowIfNull(comment);
        EnsureNotClosed();
        _comments.Add(comment);
    }

    public void Resolve()
    {
        EnsureNotClosed();
        Status = TicketStatus.Resolved;
    }

    public void Close() => Status = TicketStatus.Closed;

    public void Reopen()
    {
        if (Status is not (TicketStatus.Resolved or TicketStatus.Closed))
            throw new InvalidOperationException($"Ticket {Id} is not resolved or closed.");
        Status = Assignee is null ? TicketStatus.Open : TicketStatus.InProgress;
    }

    private void EnsureNotClosed()
    {
        if (Status == TicketStatus.Closed)
            throw new InvalidOperationException($"Ticket {Id} is closed.");
    }
}
```

The notification capability is a contract owned by the domain, implemented elsewhere:

```csharp
// src/Beacon.Core/Notifications/INotifier.cs
namespace Beacon.Core.Notifications;

public interface INotifier
{
    Task NotifyAsync(string recipient, string message, CancellationToken ct = default);
}
```

```csharp
// src/Beacon.Cli/ConsoleNotifier.cs
using Beacon.Core.Notifications;

internal sealed class ConsoleNotifier : INotifier
{
    public Task NotifyAsync(string recipient, string message, CancellationToken ct = default)
    {
        Console.WriteLine($"[notify {recipient}] {message}");
        return Task.CompletedTask;
    }
}
```

This has the shape you'll see throughout the book:

- **The domain** (`Ticket`) enforces rules and knows nothing about consoles, HTTP or
  databases.
- **Contracts** (`INotifier`) are defined where they're *needed* (the core), not where
  they're *implemented*.
- **Implementations** (`ConsoleNotifier`; later `EmailNotifier`) live at the edges.

That's the seed of the *clean architecture* and *ports and adapters* styles discussed in
Book XIII. Notice we haven't added an interface for `Ticket` itself or for `SlaRules`.
Nothing needs to substitute them.

---

## 9. What can go wrong

- **Anemic models with scattered rules.** The same validation duplicated across five
  services, slightly differently each time.
- **God classes.** A `TicketManager` with 3,000 lines that does everything. Usually a
  sign that responsibilities weren't separated (see SOLID in Book XIII).
- **Deep inheritance hierarchies.** Behavior spread over many levels; changes to a base
  class break distant subclasses.
- **Interface explosion.** Every class has an interface; navigating the code means
  constantly jumping through indirection.
- **Leaky encapsulation.** Public setters, exposed mutable collections, or `internal`
  used as a convenience rather than a boundary.
- **Over-modeling.** Strategy, factory and visitor patterns wrapped around logic that
  would be ten lines of straightforward code.

---

## 10. How an experienced engineer thinks about this

- **Start from invariants.** "What must always be true?" determines what the class's
  public operations should be.
- **Prefer composition; use inheritance sparingly and shallowly; seal by default.**
- **Abstract at real boundaries.** I/O, external services, time and randomness deserve
  interfaces. Pure logic usually doesn't.
- **Pick the paradigm that fits the problem.** Objects for stateful things with rules;
  functions and records for transformations and calculations; pattern matching for
  closed sets of cases.
- **Optimize for the reader.** The best design is the one the next developer can
  understand and change safely, often the simplest one that protects the important rules.

---

## 11. Check yourself

**Questions**

1. What is an invariant? Give two for a shopping cart.
2. Why isn't a class with a public getter and setter for every field encapsulated?
3. When does polymorphism beat a `switch`, and when does a `switch` beat polymorphism?
4. Explain the fragile base class problem.
5. What does the Liskov substitution principle require? Why is `Square : Rectangle` a
   violation with mutable properties?
6. When is an interface unnecessary?

**Exercises**

1. Add `TicketPriority` to `Ticket`, plus a method `Escalate()` that raises priority by
   one level and refuses to escalate a closed ticket or an already-urgent one.
2. Write a `switch` expression that returns the allowed next statuses for each
   `TicketStatus`. Then make `Ticket` use it to validate transitions.
3. Find a class hierarchy in a project you've worked on with three or more levels.
   Sketch how it could be rewritten using composition.

**Interview-style questions**

- "What are the four pillars of OOP? Which do you think is most important and why?"
- "Why is composition often preferred over inheritance?"
- "What's the difference between an abstract class and an interface in modern C#?"
- "What's an anemic domain model? Is it always bad?"

---

## 12. Going deeper

- *Domain-Driven Design Distilled* by Vaughn Vernon — a short introduction to modeling
  rich domains.
- Martin Fowler's articles on [Anemic Domain Model](https://martinfowler.com/bliki/AnemicDomainModel.html)
  and [Tell, Don't Ask](https://martinfowler.com/bliki/TellDontAsk.html).
- [Microsoft docs: Interfaces](https://learn.microsoft.com/dotnet/csharp/fundamentals/types/interfaces)
- Book XIII, Chapter 1 revisits these ideas through SOLID and design patterns.

**Next:** [Chapter 4 — Generics](04-generics.md) adds the other major form of
polymorphism: writing code that works for any type, safely and without boxing.
