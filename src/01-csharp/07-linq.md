# LINQ

LINQ (Language Integrated Query) is one of the features that makes C# pleasant:
filtering, transforming, grouping and joining data in a few readable lines. It's also
behind some of the most common performance problems in .NET applications: queries that
execute five times instead of once, `ToList()` calls that load an entire table into
memory, and innocent-looking `Count()` calls that walk a million elements.

All of those come from one concept: **deferred execution**. This chapter explains it
thoroughly, then covers the difference between LINQ in memory and LINQ that becomes SQL.

---

## 1. The problem: loops that hide what they mean

Here's a typical imperative loop:

```csharp
var result = new List<string>();
foreach (var t in tickets)
{
    if (t.IsActive && t.Priority >= TicketPriority.High)
        result.Add($"{t.Id}: {t.Title}");
}
result.Sort();
```

It works, but you have to read every line to understand *what* it computes. The LINQ
version states the intent directly:

```csharp
var result = tickets
    .Where(t => t.IsActive && t.Priority >= TicketPriority.High)
    .Select(t => $"{t.Id}: {t.Title}")
    .Order()
    .ToList();
```

LINQ is **declarative**: you describe the result, and the library handles the iteration.
And because the description is just method calls over `IEnumerable<T>` (or
`IQueryable<T>`), the same syntax works over lists, XML, JSON, databases and more.

---

## 2. The mental model: a pipeline of lazy iterators

### LINQ operators are extension methods

`Where`, `Select` and the rest are **extension methods** on `IEnumerable<T>`, defined in
`System.Linq.Enumerable`:

```csharp
public static IEnumerable<T> Where<T>(this IEnumerable<T> source, Func<T, bool> predicate);
```

`tickets.Where(...)` is just `Enumerable.Where(tickets, ...)`.

### Deferred execution

The crucial fact: **most LINQ operators don't do anything when you call them.** They
return an object that *describes* the operation. Work happens only when something
iterates the result.

```csharp
var query = tickets.Where(t =>
{
    Console.WriteLine($"checking {t.Id}");
    return t.IsActive;
});

Console.WriteLine("query built");      // nothing checked yet
foreach (var t in query) { }           // NOW "checking ..." prints for each ticket
foreach (var t in query) { }           // and AGAIN, for each ticket
```

How does this work? `Where` is implemented with an **iterator**: a method using
`yield return`, which the compiler turns into a state machine (a class implementing
`IEnumerable<T>` and `IEnumerator<T>`):

```csharp
static IEnumerable<T> MyWhere<T>(IEnumerable<T> source, Func<T, bool> predicate)
{
    foreach (var item in source)
        if (predicate(item))
            yield return item;   // pause here; resume on the next MoveNext()
}
```

A chain like `.Where(...).Select(...).Take(3)` builds a pipeline of iterators. When you
`foreach` over the end of it, each element is **pulled** through the whole chain, one at
a time:

```text
 foreach ──MoveNext()──► Take(3) ──MoveNext()──► Select ──MoveNext()──► Where ──MoveNext()──► List
         ◄── element ──         ◄── element ──         ◄── element ──        ◄── element ──
```

Consequences:

- **Streaming.** Elements flow one at a time. `File.ReadLines(path).Where(...).Take(10)`
  reads only as much of the file as needed to find ten matches.
- **Short-circuiting.** `Take`, `First` and `Any` stop pulling as soon as they have their
  answer.
- **Re-execution.** Every enumeration runs the whole pipeline again.
- **Late binding.** The query sees the source *as it is when iterated*, not when the query
  was built.

### Immediate vs deferred operators

| Deferred (return a lazy sequence) | Immediate (execute now) |
|---|---|
| `Where`, `Select`, `SelectMany` | `ToList`, `ToArray`, `ToDictionary`, `ToHashSet` |
| `OrderBy`, `ThenBy`* | `Count`, `Sum`, `Min`, `Max`, `Average` |
| `GroupBy`*, `Join`*, `Distinct`* | `First`, `Single`, `Last`, `ElementAt` (+ `OrDefault`) |
| `Skip`, `Take`, `Chunk`, `Zip` | `Any`, `All`, `Contains` |
| `Concat`, `Append`, `Prepend` | `Aggregate` |

\* Deferred, but must consume the *entire* source before producing the first element
(you can't know the smallest item until you've seen them all). They're "deferred but not
streaming," and they buffer everything in memory.

---

## 3. The operators you'll use most

### Filtering and projection

```csharp
var titles   = tickets.Where(t => t.IsActive).Select(t => t.Title);
var withIdx  = tickets.Select((t, index) => $"{index + 1}. {t.Title}");
var comments = tickets.SelectMany(t => t.Comments);           // flatten a sequence of sequences
var typed    = events.OfType<TicketAssigned>();               // filter by type
```

### Ordering

```csharp
var ordered = tickets
    .OrderByDescending(t => t.Priority)
    .ThenBy(t => t.CreatedAt);
```

`OrderBy` is a **stable** sort: equal elements keep their original order. Use `ThenBy`
for secondary keys; calling `OrderBy` twice discards the first ordering.

### Element access

| Method | If none | If more than one |
|---|---|---|
| `First(pred)` | throws | returns first |
| `FirstOrDefault(pred)` | `default` (null) | returns first |
| `Single(pred)` | throws | throws |
| `SingleOrDefault(pred)` | `default` | throws |

Use `Single` when there *must* be exactly one (looking up by unique key). It documents
and enforces the assumption. Use `First` when you just want any one or the first in order.

### Grouping and aggregation

```csharp
var byAssignee = tickets
    .Where(t => t.Assignee is not null)
    .GroupBy(t => t.Assignee!)
    .Select(g => new { Assignee = g.Key, Open = g.Count(t => t.IsActive), Total = g.Count() })
    .OrderByDescending(x => x.Open);

var countsByStatus = tickets.CountBy(t => t.Status);          // .NET 9+: IEnumerable<KeyValuePair<TicketStatus,int>>
var oldest = tickets.MinBy(t => t.CreatedAt);                 // .NET 6+
```

### Set operations and joins

```csharp
var unique  = names.Distinct(StringComparer.OrdinalIgnoreCase);
var byId    = tickets.DistinctBy(t => t.Id);
var both    = teamA.Intersect(teamB);
var joined  = tickets.Join(users, t => t.Assignee, u => u.Login, (t, u) => (t, u));
```

### Query syntax

C# also has SQL-like *query syntax*, which the compiler translates into the same method
calls:

```csharp
var q = from t in tickets
        where t.IsActive
        orderby t.Priority descending
        select t.Title;
```

Method syntax is more common and covers every operator. Query syntax is genuinely
clearer for joins and for queries with intermediate `let` variables.

---

## 4. `IEnumerable<T>` vs `IQueryable<T>`

This is the single most important LINQ concept for backend work.

```csharp
IEnumerable<Ticket> a = dbContext.Tickets.AsEnumerable().Where(t => t.Status == TicketStatus.Open);
IQueryable<Ticket>  b = dbContext.Tickets.Where(t => t.Status == TicketStatus.Open);
```

They look identical. They behave completely differently:

| | `IEnumerable<T>` (LINQ to Objects) | `IQueryable<T>` (LINQ providers like EF Core) |
|---|---|---|
| Lambdas become | `Func<T, bool>` delegates | `Expression<Func<T, bool>>` trees (Chapter 6) |
| Executed by | Your process, in memory | The provider: translated to SQL, run by the database |
| `Where` in `a` | Loads **every** ticket, filters in C# | — |
| `Where` in `b` | — | `SELECT ... WHERE status = 0` |

The danger is that switching from one to the other is silent. Any time a query passes
through an `IEnumerable<T>` typed variable, parameter or method (`AsEnumerable()`, a
method that returns `IEnumerable<T>`), everything after that point runs in memory.

```csharp
// Looks harmless. Loads the entire Tickets table, then filters in memory.
public IEnumerable<Ticket> GetAll() => _db.Tickets;
var open = repo.GetAll().Where(t => t.IsActive).Take(20);
```

> **⚠️ What can go wrong:** With `IQueryable<T>`, the lambda must be translatable. A C#
> method or a computed property like `t.IsActive` (defined in C#, not mapped to a column)
> can't be turned into SQL. EF Core throws *"The LINQ expression could not be
> translated."* Write the condition in terms of mapped columns, or map the computed value.
> Book IV, Chapter 7 covers this in detail.

---

## 5. Performance: what LINQ actually costs

LINQ to Objects is fast enough for nearly all business code, and the .NET team optimizes
it heavily each release (operators special-case arrays and lists, `Count()` uses
`ICollection<T>.Count` when available, and so on). But it isn't free:

- **Allocations.** Each operator in a chain allocates an iterator object; each capturing
  lambda allocates a closure and delegate (Chapter 6).
- **Indirection.** Each element passes through several `MoveNext()` calls and delegate
  invocations instead of a tight loop.

In a request handler that processes 50 items, that cost is invisible. In a loop that runs
millions of times per second, it may matter, and a plain `for` loop over a span can be
several times faster.

> **🧭 When not to use LINQ:** In hot paths identified by a profiler, in code that needs
> early exit with complex state, or when a loop with clear variable names is genuinely
> easier to read (complex multi-step logic shoved into one giant `Aggregate` is not
> clarity). Everywhere else, prefer LINQ's readability.

### Common inefficiencies

```csharp
// 1. Count() > 0 walks the whole sequence (if it's not a collection). Any() stops at the first.
if (tickets.Where(t => t.IsActive).Count() > 0) ...    // ✗
if (tickets.Any(t => t.IsActive)) ...                   // ✓

// 2. Multiple enumeration of an expensive query.
var active = LoadTickets().Where(t => t.IsActive);      // lazy
Console.WriteLine(active.Count());                      // runs it
foreach (var t in active) ...                           // runs it again
var activeList = LoadTickets().Where(t => t.IsActive).ToList();   // ✓ materialize once

// 3. Lookups inside a projection: O(n²).
var result = tickets.Select(t => users.First(u => u.Login == t.Assignee));   // ✗
var usersByLogin = users.ToDictionary(u => u.Login);                          // ✓
var result2 = tickets.Select(t => usersByLogin[t.Assignee!]);

// 4. Sorting just to take the top one.
var newest = tickets.OrderByDescending(t => t.CreatedAt).First();   // O(n log n)
var newest2 = tickets.MaxBy(t => t.CreatedAt);                      // O(n) ✓
```

---

## 6. Writing your own operators

Because LINQ operators are just extension methods over `IEnumerable<T>`, you can write
your own, and they compose with the built-ins:

```csharp
public static class TicketEnumerableExtensions
{
    public static IEnumerable<Ticket> Active(this IEnumerable<Ticket> source)
        => source.Where(t => t.IsActive);

    public static IEnumerable<Ticket> Breaching(
        this IEnumerable<Ticket> source, DateTimeOffset now)
    {
        foreach (var t in source)
        {
            if (t.IsActive && now - t.CreatedAt > SlaRules.ResponseTarget(t.Priority))
                yield return t;
        }
    }
}

var late = tickets.Active().Breaching(DateTimeOffset.UtcNow).OrderByDescending(t => t.Priority);
```

> **⚠️ What can go wrong:** Argument validation in an iterator method runs *lazily*, when
> enumeration starts, not when you call the method. If you need eager validation, split
> the method: a public non-iterator method validates and then returns the result of a
> private iterator method.

---

## 7. In practice: Beacon's reports

Beacon's managers want a CLI report: per agent, how many active tickets they have, how
many are breaching SLA, and their oldest active ticket. Let's write it with LINQ, then
review it for the problems in this chapter.

First, add the SLA rule from Chapter 3 to `Beacon.Core`:

```csharp
// src/Beacon.Core/Tickets/SlaRules.cs
namespace Beacon.Core.Tickets;

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

    public static bool IsBreaching(Ticket t, DateTimeOffset now)
        => t.IsActive && now - t.CreatedAt > ResponseTarget(t.Priority);
}
```

Then the report:

```csharp
// src/Beacon.Core/Reports/AgentWorkloadReport.cs
using Beacon.Core.Tickets;

namespace Beacon.Core.Reports;

public sealed record AgentWorkload(
    string Agent,
    int Active,
    int Breaching,
    TicketId? OldestActive);

public static class AgentWorkloadReport
{
    public static IReadOnlyList<AgentWorkload> Build(IEnumerable<Ticket> tickets, DateTimeOffset now)
    {
        return tickets
            .Where(t => t.IsActive && t.Assignee is not null)
            .GroupBy(t => t.Assignee!, StringComparer.OrdinalIgnoreCase)
            .Select(g => new AgentWorkload(
                Agent: g.Key,
                Active: g.Count(),
                Breaching: g.Count(t => SlaRules.IsBreaching(t, now)),
                OldestActive: g.MinBy(t => t.CreatedAt)?.Id))
            .OrderByDescending(w => w.Breaching)
            .ThenByDescending(w => w.Active)
            .ToList();
    }
}
```

Reviewing it:

- **Enumerated once.** `GroupBy` buffers the active, assigned tickets; each group is then
  walked a few times (`Count`, `Count(pred)`, `MinBy`), but groups are in-memory lists, so
  that's cheap.
- **Returns a materialized `IReadOnlyList<T>`.** Callers can't accidentally re-run it.
- **`now` is a parameter.** The report is a pure function: same input, same output.
  Trivial to test (Chapter 14).
- **Case-insensitive grouping**, so "Maria" and "maria" are one agent.
- **`OldestActive` is `TicketId?`**: `MinBy` returns null for an empty group (which can't
  happen here, but the types say so honestly).

Printing it in the CLI:

```csharp
foreach (var w in AgentWorkloadReport.Build(tickets, DateTimeOffset.UtcNow))
    Console.WriteLine($"{w.Agent,-10} active={w.Active,-3} breaching={w.Breaching,-3} oldest={w.OldestActive}");
```

When Book IV moves tickets into PostgreSQL, this report will run against an `IQueryable`
and need changes: `IsActive` and `SlaRules.IsBreaching` are C# logic EF Core can't
translate. That will be a good exercise in the exact problem from section 4.

---

## 8. What can go wrong

- **Multiple enumeration** of expensive or side-effecting queries.
- **Accidental in-memory evaluation**: an `IQueryable` silently becoming an
  `IEnumerable`, loading whole tables.
- **Untranslatable expressions** in EF Core queries.
- **Captured variables changing** before a deferred query runs:
  ```csharp
  var minPriority = TicketPriority.Low;
  var q = tickets.Where(t => t.Priority >= minPriority);
  minPriority = TicketPriority.Urgent;
  q.Count();   // uses Urgent
  ```
- **Hidden O(n²)** from lookups inside projections.
- **Exceptions far from their cause.** A lazy query that throws will throw where it's
  enumerated, which may be a different method, a serializer, or a view.
- **Unreadable mega-chains.** Ten operators and nested lambdas in one expression. Break
  them up with well-named intermediate variables or helper methods.

---

## 9. How an experienced engineer thinks about this

- **Always know whether a query is lazy, and where it executes.** "Is this
  `IQueryable` or `IEnumerable`?" and "When does this run?" are the two questions to ask
  of every LINQ chain in data-access code.
- **Materialize at boundaries.** Return `List<T>` / `IReadOnlyList<T>` from methods that
  fetch data; keep lazy sequences local.
- **Prefer the specific operator.** `Any` over `Count() > 0`, `MaxBy` over sort-then-first,
  `ToDictionary` once over repeated `First`.
- **Readability first, measure before rewriting.** LINQ is rarely the bottleneck; the
  database and network are.

---

## 10. Check yourself

**Questions**

1. What is deferred execution? Which operators execute immediately?
2. Why do `OrderBy` and `GroupBy` buffer the whole source even though they're deferred?
3. What does the compiler generate for a method that uses `yield return`?
4. Explain the difference between LINQ over `IEnumerable<T>` and `IQueryable<T>`.
5. Why is `Any()` usually better than `Count() > 0`?
6. When would `Single` be a better choice than `First`?

**Exercises**

1. Add `Console.WriteLine` calls inside `Where` and `Select` lambdas in a chain ending
   with `Take(2)`, and predict the output order before running it.
2. Write an iterator extension `Batch<T>(this IEnumerable<T>, int size)` (then compare
   with the built-in `Chunk`). Make argument validation eager.
3. Extend `AgentWorkloadReport` with each agent's average ticket age. Use `TimeSpan`
   arithmetic carefully.
4. Find a `foreach` loop in your own code that would be clearer as LINQ, and one LINQ
   chain that would be clearer as a loop.

**Interview-style questions**

- "What's deferred execution in LINQ? Give an example of a bug it can cause."
- "What's the difference between `IEnumerable` and `IQueryable`?"
- "How would you find the performance problem in this LINQ query?"

---

## 11. Going deeper

- [Microsoft docs: LINQ overview](https://learn.microsoft.com/dotnet/csharp/linq/)
- [Microsoft docs: Standard query operators](https://learn.microsoft.com/dotnet/csharp/linq/standard-query-operators/)
- Jon Skeet's *Edulinq* blog series, which reimplements LINQ to Objects operator by operator,
  is the best way to understand it from the inside.

**Next:** [Chapter 8 — Exceptions and Error Handling](08-exceptions-and-error-handling.md)
covers what happens when things fail, and how to design for failure deliberately.
