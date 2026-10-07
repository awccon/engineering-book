# Exceptions and Error Handling

Error handling is where experience shows most clearly. Junior code tends toward two
extremes: no handling at all, or `try { ... } catch (Exception) { }` everywhere,
swallowing problems so the application limps on in a corrupted state. Experienced code
is deliberate: it knows which failures are *expected* and which are *bugs*, handles each
at the right level, and makes sure that when something does go wrong in production,
someone can tell what happened.

---

## 1. The problem: things fail, in different ways

Consider the kinds of failure Beacon will meet:

| Failure | Example | Expected? | Who can fix it? |
|---|---|---|---|
| Invalid user input | Empty ticket title | Yes, routinely | The user |
| Business rule violation | Commenting on a closed ticket | Yes | The user |
| Missing resource | Ticket #999 doesn't exist | Yes | The user / caller |
| Transient infrastructure failure | Database timeout, HTTP 503 | Yes, occasionally | Retry later |
| Programming bug | Null dereference, index out of range | No | A developer |
| Environment failure | Out of memory, disk full, config missing | No | Operations |

These need different treatment. A bug should fail loudly and be logged with full detail.
An invalid title should produce a friendly 400 response. A database timeout might be
retried. Using one mechanism for all of them, in either direction, causes problems.

---

## 2. The mental model: what an exception really is

### Throwing and unwinding

When code throws, the runtime:

1. Creates the exception object, capturing the **stack trace** (which methods were active).
2. **Searches up the call stack** for a matching `catch` block (first pass).
3. **Unwinds** the stack to that point, running every `finally` block on the way (second pass).
4. Runs the `catch` block.

If no handler is found, the exception is **unhandled**: the runtime logs it and
terminates the process.

```text
  Main()                        ◄── catch (InvalidOperationException) here
   └─ TicketService.Resolve()       finally { ... } runs during unwinding
       └─ Ticket.Resolve()
           └─ EnsureNotClosed()  ── throw new InvalidOperationException(...)
```

### Exceptions are expensive (relative to normal code)

Throwing captures a stack trace, searches handlers, and unwinds frames. It's thousands of
times slower than returning a value. .NET 9 made exception handling substantially faster,
but the principle stands: **exceptions are for exceptional situations**, not normal
control flow. A login form that throws for every wrong password, or a parser that throws
for every invalid line, puts that cost on a common path.

### The exception hierarchy

```text
System.Exception
 ├─ SystemException
 │   ├─ ArgumentException
 │   │   ├─ ArgumentNullException
 │   │   └─ ArgumentOutOfRangeException
 │   ├─ InvalidOperationException
 │   │   └─ ObjectDisposedException
 │   ├─ NullReferenceException          (a bug: never throw or catch this)
 │   ├─ IndexOutOfRangeException        (a bug)
 │   ├─ NotSupportedException
 │   ├─ FormatException
 │   ├─ IOException
 │   ├─ OperationCanceledException
 │   │   └─ TaskCanceledException
 │   └─ OutOfMemoryException, StackOverflowException  (can't meaningfully recover)
 ├─ HttpRequestException
 └─ your own exceptions
```

Know the standard ones and use them as intended:

- **`ArgumentException` (and subclasses):** the *caller* passed something invalid. A bug in
  the caller.
- **`InvalidOperationException`:** the call isn't valid in the object's *current state*
  (resolving a closed ticket, reading from a closed reader).
- **`NotSupportedException`:** the operation is never supported by this type.
- **`OperationCanceledException`:** a `CancellationToken` was triggered. Not an error;
  usually let it propagate.

---

## 3. Throwing well

### Use the guard helpers

Modern .NET provides static throw helpers that are concise and produce good messages:

```csharp
ArgumentNullException.ThrowIfNull(ticket);
ArgumentException.ThrowIfNullOrWhiteSpace(title);
ArgumentOutOfRangeException.ThrowIfNegative(count);
ArgumentOutOfRangeException.ThrowIfGreaterThan(pageSize, 100);
ObjectDisposedException.ThrowIf(_disposed, this);
```

They use `[CallerArgumentExpression]` to include the argument name automatically.

### Write messages for the person debugging at 3 a.m.

```csharp
throw new InvalidOperationException("Invalid state.");                                // ✗
throw new InvalidOperationException($"Ticket {Id} can't be resolved because it is {Status}."); // ✓
```

Include identifiers and the actual state. Never include secrets, passwords, tokens or
personal data: exception messages end up in logs.

### Preserve the original exception

```csharp
catch (Exception ex)
{
    throw;            // ✓ rethrows, preserving the original stack trace
    throw ex;         // ✗ resets the stack trace to this line; you lose where it really failed
}

catch (NpgsqlException ex)
{
    throw new TicketStoreException($"Failed to save ticket {id}.", ex);   // ✓ wrap with InnerException
}
```

Wrapping is useful when translating low-level failures into a concept the caller
understands, but always pass the original as `innerException`.

### Custom exceptions

Create a custom exception type when callers need to **catch it specifically** and
handle it differently. If no one will ever catch it by type, a standard exception with
a good message is enough.

```csharp
public sealed class TicketClosedException(TicketId id)
    : InvalidOperationException($"Ticket {id} is closed.")
{
    public TicketId TicketId { get; } = id;
}
```

---

## 4. Catching well

### Catch only what you can handle

The rule: catch an exception **only if you can do something useful about it** at that
point:

- **Recover**: retry, use a fallback, return a default that's genuinely correct.
- **Translate**: convert to a different exception or result meaningful at this layer.
- **Add context**: log or wrap with information only this level knows, then rethrow.
- **Stop the process cleanly** at the very top level.

Otherwise, let it propagate. Most methods should contain *no* `try/catch` at all.

### Never swallow silently

```csharp
try { SendEmail(); }
catch { }               // ✗ the email failed, nobody knows, nobody will ever know
```

If failure is genuinely acceptable (a best-effort notification), at minimum log it at an
appropriate level. Code like the example above turns bugs into mysteries.

### Be specific, and use filters

```csharp
try
{
    return await client.GetFromJsonAsync<TicketDto>(url, ct);
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    return null;     // 404 is an expected answer here
}
```

**Exception filters** (`when (...)`) decide whether to catch *without unwinding the
stack* if the condition is false. That means the original stack is intact for debuggers
and crash dumps, and it's cleaner than catching and rethrowing.

### `catch (Exception)` belongs at boundaries

Catching all exceptions is correct in exactly a few places, the **boundaries** of the
application:

- the top of a request pipeline (ASP.NET Core's exception handling middleware does this),
- the top of a background job loop (so one failed message doesn't kill the worker),
- `Main` (to log and exit with a non-zero code).

There, the handler logs the full exception and converts it into an appropriate response
(HTTP 500, a dead-lettered message, an exit code).

### `finally` and `using`

`finally` blocks run whether the `try` completes normally or throws. They're for cleanup.
In practice, you use `using` for most cleanup, because it generates the `try/finally`:

```csharp
await using var connection = await dataSource.OpenConnectionAsync(ct);
// connection is disposed at the end of the scope, even if an exception is thrown
```

---

## 5. Exceptions vs results: the great debate

There are two broad styles for expected failures:

**Exceptions:** `Ticket.Resolve()` throws if the ticket is closed.

**Result values:** `Ticket.Resolve()` returns `Result<Ticket>` (Chapter 4), and the
caller checks it.

| | Exceptions | Result types |
|---|---|---|
| Can callers forget to handle it? | Yes, but it propagates loudly | Yes, but the type is right there; `Match` forces it |
| Cost | Expensive when thrown | Cheap |
| Visible in signatures | No | Yes |
| Plays well with the BCL and frameworks | Native | Needs adapters at boundaries |
| Boilerplate | Little | More (checking and propagating) |

A pragmatic rule many experienced teams converge on:

- **Bugs and broken invariants → exceptions.** A null argument, an impossible state, a
  violated contract. These should be loud, and no caller can meaningfully "handle" them.
- **Expected business outcomes → results** (or `TryX` patterns) *at the application
  boundary*: "not found," "validation failed," "conflict." These are normal answers, and
  you'll want to map them to HTTP status codes, not 500s.
- **Infrastructure failures → exceptions**, handled by retry policies or boundary
  handlers.

The `Try` pattern is the BCL's lightweight version of a result:

```csharp
if (int.TryParse(input, out var n)) { ... }
if (cache.TryGetValue(key, out var value)) { ... }
```

> **🧭 When not to use result types:** Don't force results through every layer. If every
> private method returns `Result<T>`, you've reinvented exceptions with more typing. Use
> results where the outcome is a meaningful, expected answer that the caller will branch
> on, typically at the application-service level.

---

## 6. Exceptions and async code

Exceptions in async methods are captured in the returned `Task` and rethrown when you
`await` it, with the original stack trace preserved. That's natural and mostly works as
you'd expect. Three caveats (Chapter 9 covers them in depth):

- **`async void` methods** have no task to put the exception in, so exceptions are rethrown
  on the synchronization context or thread pool, which usually **crashes the process**.
  Only use `async void` for event handlers.
- **`Task.WhenAll`** throws only the *first* exception when awaited; the others are in
  `task.Exception.InnerExceptions`.
- **Unobserved tasks** (fire-and-forget) swallow exceptions unless something observes
  them.

### Cancellation is not failure

When a `CancellationToken` is triggered, async operations throw
`OperationCanceledException`. If the caller requested cancellation (a user closed the
browser, the app is shutting down), that's not an error and shouldn't be logged as one:

```csharp
catch (OperationCanceledException) when (ct.IsCancellationRequested)
{
    // expected: the caller cancelled
}
```

---

## 7. Production error handling: logging and observability

Handling an error in production is only half the job; the other half is being able to
**diagnose** it later.

- **Log once, at the boundary.** If every layer catches, logs and rethrows, one failure
  produces five log entries. Log at the place that finally handles it.
- **Log the exception object, not just the message.** `logger.LogError(ex, "Failed to
  resolve ticket {TicketId}", id)` keeps the type, stack trace and inner exceptions.
- **Use structured logging** with named placeholders, as above, so you can search for
  every failure on ticket `T-42`. Book III, Chapter 3 covers this.
- **Correlate.** Every log line in a request should carry a trace ID, so you can follow one
  failure across services (Book IX, Chapter 6).
- **Never leak internals to users.** Stack traces and SQL in an API response are a security
  problem. Return a generic message plus a correlation ID; log the details.

> **🔍 Investigation: reading a stack trace.** Read the exception type and message first,
> then find the *top-most frame in your own code*, not framework code. For wrapped
> exceptions, scroll to the innermost one (`---> ` lines); that's usually the real cause.
> For async code, frames like `--- End of stack trace from previous location ---` mark
> where the exception crossed an `await`; read through them.

---

## 8. In practice: Beacon's error strategy

Let's apply a clear policy to Beacon:

1. **Domain invariants** (`Ticket`) throw. Calling `AddComment` on a closed ticket is a
   programming error *if the application layer allowed it*.
2. **Application services** check expected conditions first and return `Result<T>` for
   expected failures (not found, rule violations the user can trigger).
3. **The boundary** (CLI now, API later) maps results to output and catches everything
   else, logging it.

### A domain exception type

```csharp
// src/Beacon.Core/Common/DomainException.cs
namespace Beacon.Core.Common;

/// <summary>A domain rule was violated. Indicates the caller didn't check a precondition.</summary>
public sealed class DomainException(string message) : InvalidOperationException(message);
```

`Ticket.EnsureNotClosed()` now throws `DomainException`.

### Application service returning results

```csharp
// src/Beacon.Core/Tickets/TicketService.cs
using Beacon.Core.Common;

namespace Beacon.Core.Tickets;

public interface ITicketRepository
{
    Task<Ticket?> FindAsync(TicketId id, CancellationToken ct = default);
    Task SaveAsync(Ticket ticket, CancellationToken ct = default);
}

public sealed class TicketService(ITicketRepository tickets, TimeProvider clock)
{
    public async Task<Result<Ticket>> AddCommentAsync(
        TicketId id, string author, string body, CancellationToken ct = default)
    {
        if (string.IsNullOrWhiteSpace(body))
            return Error.Validation("Comment body is required.");

        var ticket = await tickets.FindAsync(id, ct);
        if (ticket is null)
            return Error.NotFound($"Ticket {id}");

        if (ticket.Status == TicketStatus.Closed)
            return Error.Conflict($"Ticket {id} is closed and can't receive comments.");

        ticket.AddComment(new Comment(author, body.Trim(), clock.GetUtcNow()));
        await tickets.SaveAsync(ticket, ct);
        return ticket;
    }
}
```

Note the layering: the service checks *expected* conditions and returns friendly errors.
The domain's own check (`EnsureNotClosed` throwing) is still there as a safety net. If a
future developer calls `AddComment` from somewhere that forgot to check, the invariant
still holds, and the bug is loud.

`TimeProvider` (.NET 8+) is the BCL's abstraction over the clock, so tests can control
time.

### The boundary

```csharp
// src/Beacon.Cli/Program.cs (excerpt)
try
{
    var result = await service.AddCommentAsync(new TicketId(7), "maria", "Restarted the VPN gateway.");
    Console.WriteLine(result.Match(
        t => $"Comment added to {t.Id}.",
        e => $"Could not add comment ({e.Code}): {e.Message}"));
    return 0;
}
catch (OperationCanceledException)
{
    Console.Error.WriteLine("Cancelled.");
    return 130;
}
catch (Exception ex)
{
    Console.Error.WriteLine($"Unexpected error: {ex}");   // full details: type, message, stack
    return 1;
}
```

Expected failures produce a clear message and success exit code; unexpected ones are
reported in full and exit non-zero (which matters when the CLI runs in scripts or CI).
When Beacon gets an API in Book III, the same `Error.Code` values will map to HTTP status
codes: `not_found` → 404, `validation` → 400, `conflict` → 409.

---

## 9. What can go wrong

- **Swallowed exceptions** (`catch { }`) hiding real failures.
- **`throw ex;`** destroying the stack trace.
- **Catch-log-rethrow at every layer**, producing noisy, duplicated logs.
- **Exceptions for control flow** on hot paths (parsing, validation, "not found").
- **Catching `Exception` deep in the code** and continuing with corrupted state.
- **Leaking details** (stack traces, SQL, internal IDs, personal data) to users.
- **Ignoring cancellation**, logging every cancelled request as an error.
- **`async void`** crashing the process.
- **Missing context**: "Object reference not set to an instance of an object" with no
  identifiers.

---

## 10. How an experienced engineer thinks about this

- **Classify the failure first.** Bug, expected outcome, transient infrastructure or
  environment. The category decides the mechanism.
- **Fail fast on bugs.** A process that crashes loudly on a broken invariant is better
  than one that continues with corrupted data.
- **Handle at the level that has enough context.** Usually that's either right where
  the failure happens (retry, fallback) or at the boundary (log, respond).
- **Design errors as part of the API.** What can go wrong is as much a part of a
  method's contract as what it returns.
- **Think about the person diagnosing it.** Every exception message and log entry is a
  message to a future engineer under pressure, possibly you.

---

## 11. Check yourself

**Questions**

1. What happens, step by step, when an exception is thrown?
2. What's the difference between `throw;` and `throw ex;`?
3. When is it correct to catch `Exception`?
4. What do exception filters do, and why is that better than catch-and-rethrow?
5. Which failures should be exceptions and which should be result values? Give an example
   of each.
6. Why shouldn't `OperationCanceledException` usually be logged as an error?

**Exercises**

1. Add an `Escalate` method to `TicketService` returning `Result<Ticket>`, with
   not-found, already-urgent and closed cases.
2. Write a method that throws inside a nested call, catch it with `throw ex;` and with
   `throw;`, and compare the stack traces.
3. Write `async void` code that throws, run it in a console app, and observe the crash.
   Then fix it.
4. Review a project you've worked on for empty `catch` blocks and log-and-rethrow chains.

**Interview-style questions**

- "How do you decide between throwing an exception and returning an error?"
- "What's wrong with catching `Exception` everywhere?"
- "How do you handle errors globally in an ASP.NET Core application?"

---

## 12. Going deeper

- [Microsoft docs: Best practices for exceptions](https://learn.microsoft.com/dotnet/standard/exceptions/best-practices-for-exceptions)
- [Microsoft docs: Exception handling in async code](https://learn.microsoft.com/dotnet/csharp/asynchronous-programming/)
- Eric Lippert's article *Vexing exceptions*, a classic classification of exceptions into
  fatal, boneheaded, vexing and exogenous.

**Next:** [Chapter 9 — Async/Await from the Inside](09-async-await-from-the-inside.md)
takes apart the feature every modern .NET application depends on.
