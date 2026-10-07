# Delegates, Events and Lambdas

Every time you write `tickets.Where(t => t.IsActive)`, register a minimal API endpoint
with `app.MapGet("/tickets", () => ...)`, or subscribe to a button click, you're passing
a function as a value. C# makes this feel effortless, but under the surface are
**delegates**, **closures** and compiler-generated classes, and that machinery explains
some classic bugs: the event subscription that leaks memory, the loop variable that's
captured "wrong," and the lambda that allocates on every call in a hot path.

---

## 1. The problem: passing behavior, not just data

Many problems have a fixed structure with a variable step:

- **Filter** a list, but the condition varies.
- **Retry** an operation, but the operation varies.
- **Notify** interested parties when something happens, but who's interested varies.

Without a way to pass behavior, you'd need an interface and a class for every variation:
`IFilterCondition` with `ActiveTicketsCondition`, `HighPriorityCondition`, and so on.
A **delegate** is a type-safe reference to a method, so behavior can be passed, stored
and invoked like any other value.

---

## 2. The mental model: a delegate is an object that points at a method

```csharp
public delegate bool TicketFilter(Ticket ticket);

static bool IsUrgent(Ticket t) => t.Priority == TicketPriority.Urgent;

TicketFilter filter = IsUrgent;      // create a delegate pointing at IsUrgent
bool result = filter(someTicket);    // invoke it
```

Under the hood, a delegate is a **heap object** (a class deriving from
`System.MulticastDelegate`) holding two things:

```text
  TicketFilter delegate object
  ┌──────────────────────────────┐
  │ _target  → object instance   │  (null for static methods)
  │ _methodPtr → IsUrgent code   │
  │ invocation list (multicast)  │
  └──────────────────────────────┘
```

Invoking the delegate calls the method on the target. It's essentially a single-method
interface that the runtime implements for you.

### Built-in delegate types

You rarely declare delegate types anymore. The BCL provides generic ones:

| Type | Signature | Example |
|---|---|---|
| `Action` | `() → void` | `Action log = () => Console.WriteLine("hi");` |
| `Action<T1, …>` | `(T1, …) → void` | `Action<string> print = Console.WriteLine;` |
| `Func<TResult>` | `() → TResult` | `Func<DateTimeOffset> now = () => DateTimeOffset.UtcNow;` |
| `Func<T1, …, TResult>` | `(T1, …) → TResult` | `Func<Ticket, bool> isActive = t => t.IsActive;` |
| `Predicate<T>` | `T → bool` | Older APIs like `List<T>.FindAll` |

Custom delegate types are still useful when you want a meaningful name in a public API
(`RequestDelegate` in ASP.NET Core is a good example) or need `ref`/`out` parameters.

---

## 3. Lambdas and closures

### Lambda syntax

A **lambda expression** is an anonymous function:

```csharp
Func<int, int> square = x => x * x;                 // expression body
Func<int, int, int> add = (a, b) => a + b;
Func<Ticket, string> describe = t =>
{
    var who = t.Assignee ?? "nobody";
    return $"{t.Id} assigned to {who}";              // statement body
};
```

Since C# 10, lambdas can have a *natural type*, so `var` works for simple cases:

```csharp
var parse = (string s) => int.Parse(s);   // inferred as Func<string, int>
```

### Closures: capturing variables

Lambdas can use variables from the surrounding scope:

```csharp
int threshold = 3;
Func<Ticket, bool> tooManyComments = t => t.Comments.Count > threshold;
threshold = 10;
// tooManyComments now uses 10, not 3
```

The lambda captures the **variable**, not its value at creation time. How? The compiler
rewrites the code. Roughly:

```csharp
// What the compiler generates (simplified)
sealed class DisplayClass
{
    public int threshold;
    public bool Lambda(Ticket t) => t.Comments.Count > threshold;
}

var closure = new DisplayClass();
closure.threshold = 3;
Func<Ticket, bool> tooManyComments = closure.Lambda;
closure.threshold = 10;
```

The local variable `threshold` is *hoisted* into a field of a compiler-generated class.
Both your method and the lambda now use that field. This is called a **closure**.

Consequences:

1. **Allocation.** Creating a capturing lambda allocates a closure object *and* a delegate.
   A non-capturing lambda (`t => t.IsActive`) is cached by the compiler in a static field
   and allocates nothing after the first use.
2. **Lifetime extension.** Captured variables live as long as the delegate does. If the
   delegate is stored somewhere long-lived, everything it captured stays alive too.
3. **Shared state.** Multiple lambdas capturing the same variable share it.

### `static` lambdas

If you want the compiler to guarantee you're not capturing anything, mark the lambda
`static`:

```csharp
var active = tickets.Where(static t => t.IsActive);
// static t => t.Comments.Count > threshold   → compile error: can't capture 'threshold'
```

Useful in hot paths where an accidental capture would cause allocations on every call.

### The loop capture bug (and its fix)

A famous bug, fixed for `foreach` in C# 5 but still present for `for`:

```csharp
var actions = new List<Action>();
for (int i = 0; i < 3; i++)
    actions.Add(() => Console.WriteLine(i));

foreach (var a in actions) a();   // prints 3, 3, 3
```

All three lambdas capture the *same* variable `i`, which is 3 by the time they run.
`foreach` declares a fresh variable per iteration (since C# 5), so the equivalent
`foreach` code prints 0, 1, 2. With `for`, copy into a local inside the loop:

```csharp
for (int i = 0; i < 3; i++)
{
    int copy = i;
    actions.Add(() => Console.WriteLine(copy));
}
```

---

## 4. Higher-order functions

A **higher-order function** takes or returns functions. It's the main way delegates
reduce duplication:

```csharp
public static async Task<T> RetryAsync<T>(
    Func<CancellationToken, Task<T>> operation,
    int attempts,
    TimeSpan delay,
    CancellationToken ct)
{
    for (int attempt = 1; ; attempt++)
    {
        try
        {
            return await operation(ct);
        }
        catch (Exception) when (attempt < attempts)
        {
            await Task.Delay(delay * attempt, ct);   // linear back-off
        }
    }
}

var ticket = await RetryAsync(ct => api.GetTicketAsync(id, ct), attempts: 3,
                              delay: TimeSpan.FromMilliseconds(200), ct);
```

The retry policy is written once; the operation varies. (For production retries, use a
library like Polly or `Microsoft.Extensions.Http.Resilience`; Book III covers them.)

Functions can also *return* functions, for example to build a configured filter:

```csharp
static Func<Ticket, bool> AssignedTo(string person) => t => t.Assignee == person;

var mine = tickets.Where(AssignedTo("maria"));
```

---

## 5. Events: the observer pattern built into the language

An **event** lets an object announce that something happened, without knowing who's
listening. It's the **observer pattern**, built on multicast delegates.

```csharp
public sealed class TicketBoard
{
    public event EventHandler<TicketChangedEventArgs>? TicketChanged;

    public void Resolve(Ticket ticket)
    {
        ticket.Resolve();
        TicketChanged?.Invoke(this, new TicketChangedEventArgs(ticket.Id, ticket.Status));
    }
}

public sealed class TicketChangedEventArgs(TicketId id, TicketStatus status) : EventArgs
{
    public TicketId Id { get; } = id;
    public TicketStatus Status { get; } = status;
}
```

Subscribers attach and detach with `+=` and `-=`:

```csharp
board.TicketChanged += OnTicketChanged;
board.TicketChanged -= OnTicketChanged;

void OnTicketChanged(object? sender, TicketChangedEventArgs e)
    => Console.WriteLine($"{e.Id} is now {e.Status}");
```

### What `event` adds over a delegate field

You could expose a public delegate field instead, but then any code could:

- **invoke it** (raising the event on the object's behalf),
- **replace it** with `=` (silently removing everyone else's subscriptions).

The `event` keyword restricts outside code to `+=` and `-=`. Only the declaring class
can raise the event.

### Multicast delegates

A delegate can hold an **invocation list** of several methods. `+=` adds to it; invoking
calls each in order. If one throws, later ones don't run, and if the delegate returns a
value, you only get the last one's. That's why event handlers return `void` and why
robust publishers sometimes iterate `GetInvocationList()` with individual `try/catch`.

### The event memory leak

This is the most important practical lesson in this chapter:

```text
  Long-lived publisher                 Short-lived subscriber
  ┌────────────────────┐               ┌──────────────────────┐
  │ TicketBoard        │               │ TicketDetailsView    │
  │ TicketChanged ─────┼──delegate────►│ (should be collected │
  │  (lives forever)   │   _target     │  when closed)        │
  └────────────────────┘               └──────────────────────┘
```

When a subscriber does `board.TicketChanged += this.Handler`, the publisher's delegate
holds a reference to the subscriber. If the publisher lives longer (a singleton service,
a static event), the subscriber can **never be garbage collected** until it unsubscribes.
In desktop apps this was the classic leak: closed windows that stayed in memory forever.
In server apps, it's a singleton holding a reference to every per-request object that
ever subscribed.

*Fixes:* always unsubscribe (implement `IDisposable` and unsubscribe in `Dispose`),
avoid static events, or use a different mechanism (a message bus, `IObservable<T>`, or
`Channel<T>`) when publishers and subscribers have different lifetimes.

> **🧭 When not to use events:** C# events are synchronous, in-process and tightly
> coupled to object lifetimes. They suit UI components and in-process notifications
> within one object graph. For "when a ticket is resolved, send an email and update
> analytics" in a server app, use explicit domain events dispatched by a mediator or
> outbox (Book XIII, Chapter 4), so the work can be async, retried and observed.

---

## 6. Expression trees: code as data

There's one more twist. The same lambda can compile to two different things depending on
the target type:

```csharp
Func<Ticket, bool> compiled = t => t.IsActive;                    // a delegate: executable code
Expression<Func<Ticket, bool>> tree = t => t.IsActive;            // a data structure describing the code
```

An **expression tree** is an object graph representing the lambda's structure:
"parameter `t`, member access `IsActive`." Code can *inspect* it, which is exactly how
EF Core translates `Where(t => t.Status == TicketStatus.Open)` into
`WHERE status = 0` in SQL.

This distinction is behind `IEnumerable<T>` vs `IQueryable<T>` (Chapter 7): LINQ over
`IEnumerable` takes `Func` delegates and runs them in memory; LINQ over `IQueryable`
takes `Expression` trees and translates them into something else.

> **⚠️ What can go wrong:** Expression trees can only represent a subset of C#. A method
> EF Core doesn't understand (`t => MyHelper(t)`) can't be translated to SQL, and EF Core
> throws "could not be translated." You'll meet this in Book IV.

---

## 7. Functional style in C#

Delegates, lambdas, records and pattern matching together let you write in a largely
**functional** style where it helps:

- **Pure functions**: output depends only on input; no side effects. Easy to test and
  reason about.
- **Immutable data**: records and `with` expressions.
- **Composition**: building behavior from small functions.

```csharp
static Func<T, bool> And<T>(Func<T, bool> a, Func<T, bool> b) => x => a(x) && b(x);

var urgentAndUnassigned = And<Ticket>(
    t => t.Priority == TicketPriority.Urgent,
    t => t.Assignee is null);
```

C# isn't F# or Haskell, and forcing a purely functional style on it produces awkward code.
The practical approach: functional style for transformations and rules, objects for
stateful things with identity.

---

## 8. In practice: domain events in Beacon

Beacon needs to react when tickets change: notify the assignee, update an activity feed,
and later update a search index. We don't want `Ticket` to know about any of those.

A simple, explicit approach used in many production systems: the entity *records* events
as data, and the application layer *dispatches* them after the change succeeds.

```csharp
// src/Beacon.Core/Common/IDomainEvent.cs
namespace Beacon.Core.Common;

public interface IDomainEvent
{
    DateTimeOffset OccurredAt { get; }
}
```

```csharp
// src/Beacon.Core/Tickets/TicketEvents.cs
using Beacon.Core.Common;

namespace Beacon.Core.Tickets;

public sealed record TicketAssigned(TicketId TicketId, string Assignee, DateTimeOffset OccurredAt) : IDomainEvent;
public sealed record TicketResolved(TicketId TicketId, DateTimeOffset OccurredAt) : IDomainEvent;
```

Add an event list to `Ticket`:

```csharp
private readonly List<IDomainEvent> _events = [];
public IReadOnlyList<IDomainEvent> DomainEvents => _events;
public void ClearDomainEvents() => _events.Clear();

public void AssignTo(string assignee, DateTimeOffset now)
{
    ArgumentException.ThrowIfNullOrWhiteSpace(assignee);
    EnsureNotClosed();
    Assignee = assignee;
    if (Status == TicketStatus.Open) Status = TicketStatus.InProgress;
    _events.Add(new TicketAssigned(Id, assignee, now));
}
```

(`Resolve` gets the same treatment. Methods now take `now` as a parameter, so the domain
doesn't read the clock itself, which makes it deterministic and testable. Chapter 14
returns to this.)

A tiny dispatcher maps event types to handlers, using delegates:

```csharp
// src/Beacon.Core/Common/EventDispatcher.cs
namespace Beacon.Core.Common;

public sealed class EventDispatcher
{
    private readonly Dictionary<Type, List<Func<IDomainEvent, CancellationToken, Task>>> _handlers = new();

    public void On<TEvent>(Func<TEvent, CancellationToken, Task> handler) where TEvent : IDomainEvent
    {
        if (!_handlers.TryGetValue(typeof(TEvent), out var list))
            _handlers[typeof(TEvent)] = list = [];
        list.Add((e, ct) => handler((TEvent)e, ct));   // adapt typed handler to the general shape
    }

    public async Task DispatchAsync(IEnumerable<IDomainEvent> events, CancellationToken ct = default)
    {
        foreach (var e in events)
        {
            if (!_handlers.TryGetValue(e.GetType(), out var list)) continue;
            foreach (var handler in list)
                await handler(e, ct);
        }
    }
}
```

Wiring it up in the CLI:

```csharp
var notifier = new ConsoleNotifier();
var dispatcher = new EventDispatcher();

dispatcher.On<TicketAssigned>((e, ct) =>
    notifier.NotifyAsync(e.Assignee, $"You were assigned {e.TicketId}.", ct));
dispatcher.On<TicketResolved>((e, _) =>
{
    Console.WriteLine($"[feed] {e.TicketId} resolved at {e.OccurredAt:t}");
    return Task.CompletedTask;
});

var ticket = new Ticket(new TicketId(7), "VPN drops every hour", TicketPriority.High, DateTimeOffset.UtcNow);
ticket.AssignTo("omar", DateTimeOffset.UtcNow);
ticket.Resolve(DateTimeOffset.UtcNow);

await dispatcher.DispatchAsync(ticket.DomainEvents);
ticket.ClearDomainEvents();
```

```text
[notify omar] You were assigned T-7.
[feed] T-7 resolved at 14:32
```

Why this design rather than C# `event`s on `Ticket`?

- **No lifetime coupling.** Tickets hold events as *data*, not references to subscribers.
  Nothing leaks.
- **Dispatch happens after success.** In Book IV, events will be dispatched only after the
  database transaction commits (or saved in an *outbox*), so we never notify about a
  change that was rolled back.
- **Handlers are async**, which C# events don't support well.

The lambda passed to `list.Add` is a closure that captures `handler`. That's the
adapter pattern, written in one line.

---

## 9. What can go wrong

- **Event subscription leaks** (section 5).
- **Captured variables changing unexpectedly**, especially in `for` loops or when a
  lambda outlives the method that created it.
- **Hidden allocations.** Capturing lambdas in hot paths allocate a closure and a delegate
  per call. Use `static` lambdas or pass state explicitly (many APIs offer overloads that
  take a `state` argument for exactly this reason).
- **Exceptions in multicast delegates** stop later handlers from running.
- **`async void` lambdas.** Passing an async lambda to an `Action` parameter creates an
  `async void` method; exceptions crash the process and you can't await it. Chapter 9
  explains.
- **Untranslatable expression trees** in EF Core queries.

---

## 10. How an experienced engineer thinks about this

- **Delegates for small, local variation; interfaces for larger contracts.** A
  `Func<Ticket, bool>` filter is clearer than an `ITicketFilter` interface. A notifier
  with several methods and dependencies deserves an interface.
- **Mind lifetimes.** Whenever a delegate is stored, ask what it captures and how long
  it will live.
- **Prefer explicit events-as-data** for domain notifications in server code; keep C#
  `event`s for UI and tight in-process scenarios.
- **Know which lambdas run where.** A lambda passed to `IQueryable` becomes SQL; a lambda
  passed to `IEnumerable` runs in your process. That one distinction prevents many EF
  Core performance disasters.

---

## 11. Check yourself

**Questions**

1. What two pieces of information does a delegate object hold?
2. What does the compiler generate when a lambda captures a local variable?
3. Why do non-capturing lambdas not allocate on each use?
4. What does the `event` keyword prevent that a public delegate field allows?
5. Explain how an event subscription can cause a memory leak.
6. What's the difference between `Func<Ticket, bool>` and
   `Expression<Func<Ticket, bool>>`?

**Exercises**

1. Reproduce the `for` loop capture bug, then fix it. Decompile both versions in SharpLab
   and find the display class.
2. Add `Or<T>` and `Not<T>` combinators next to `And<T>` and build a filter for
   "urgent or unassigned, but not closed."
3. Make `EventDispatcher.DispatchAsync` run every handler even if one throws, then throw
   an `AggregateException` at the end containing all failures.
4. Write a small class that subscribes to a static event and is never collected. Use
   `WeakReference` and `GC.Collect()` to demonstrate the leak, then fix it.

**Interview-style questions**

- "What's a closure? How does C# implement it?"
- "What's the difference between a delegate and an event?"
- "How can event handlers cause memory leaks in .NET?"
- "Why does EF Core need expression trees?"

---

## 12. Going deeper

- [Microsoft docs: Delegates and events](https://learn.microsoft.com/dotnet/csharp/delegates-overview)
- [Microsoft docs: Lambda expressions](https://learn.microsoft.com/dotnet/csharp/language-reference/operators/lambda-expressions)
- [Microsoft docs: Expression trees](https://learn.microsoft.com/dotnet/csharp/advanced-topics/expression-trees/)

**Next:** [Chapter 7 — LINQ](07-linq.md) builds directly on delegates and expression
trees to give you a query language inside C#.
