# Async/Await from the Inside

`async` and `await` are so convenient that most developers use them correctly most of
the time without knowing what they do. The rest of the time produces some of the hardest
production problems in .NET: deadlocks that only happen under load, thread-pool
starvation that makes a healthy server stop responding, and exceptions that vanish
without a trace.

This chapter opens the box. Once you see the state machine the compiler generates and
understand what a `Task` really is, the rules for async code stop being rules to
memorize and become obvious consequences.

---

## 1. The problem: waiting wastes threads

A typical web request spends most of its time **waiting**: for the database, for another
HTTP service, for a file. The CPU work is often a few milliseconds; the waiting can be
hundreds.

With **synchronous** code, the thread handling the request sits blocked during every
wait:

```csharp
var ticket = repository.Find(id);         // thread blocked ~20 ms waiting for the database
var user   = userApi.GetUser(ticket.Assignee);  // thread blocked ~80 ms waiting for HTTP
```

Threads are expensive: each has a stack (typically 1 MB reserved), and the operating
system must schedule them. A server with 200 threads, each blocked 95% of the time, can
only handle about 200 concurrent requests, while the CPU sits mostly idle.

**Asynchronous** code releases the thread while waiting. The thread goes back to the pool
to serve other requests; when the I/O completes, *some* thread picks up where the
operation left off. The same server can handle thousands of concurrent requests with a
handful of threads.

> **🧱 Durable:** Async is about **scalability**, not speed. A single async request isn't
> faster than a synchronous one (it's slightly slower, because of bookkeeping). Async
> lets one machine handle far more *concurrent* waiting operations.

---

## 2. The mental model

### A `Task` is a promise of a future result

A `Task<T>` represents an operation that will complete at some point, successfully with a
`T`, with an exception, or cancelled. It's an object you can:

- check (`IsCompleted`, `Status`),
- attach a **continuation** to ("when you finish, run this"),
- and `await`.

Crucially, **a `Task` is not a thread.** Most I/O tasks have no thread at all while
pending. When you call `ReadAsync` on a socket, .NET asks the operating system to notify
it when data arrives (via I/O completion ports on Windows, epoll on Linux, kqueue on
macOS) and returns an incomplete task. No thread waits. When the OS signals completion,
a thread-pool thread completes the task and runs its continuations.

```text
 Thread-pool thread A                    OS / network              Thread-pool thread B
 ─────────────────────                   ────────────              ─────────────────────
 FindAsync(id) starts
 sends query ──────────────────────────► (waiting for DB)
 hits await, method returns
 A goes back to the pool                                   
 (serves other requests)                 DB responds ─────────────► completes the Task,
                                                                     resumes FindAsync
                                                                     after the await
```

### `await` splits your method into pieces

When you write:

```csharp
public async Task<string> DescribeAsync(TicketId id, CancellationToken ct)
{
    var ticket = await repository.FindAsync(id, ct);
    var user = await users.GetAsync(ticket!.Assignee!, ct);
    return $"{ticket.Title} — {user.DisplayName}";
}
```

the compiler rewrites it into a **state machine**: a struct with a `MoveNext()` method,
fields for every local variable that lives across an `await`, and an integer `state`
recording where to resume. Simplified:

```csharp
struct DescribeAsyncStateMachine : IAsyncStateMachine
{
    public int state;                                 // -1 = running, 0/1 = awaiting #1/#2
    public AsyncTaskMethodBuilder<string> builder;    // produces the Task returned to the caller
    public TicketId id; public CancellationToken ct;  // parameters
    private Ticket? ticket;                           // locals that survive awaits
    private TaskAwaiter<Ticket?> awaiter1;
    private TaskAwaiter<User> awaiter2;

    public void MoveNext()
    {
        try
        {
            if (state == 0) goto Resume1;
            if (state == 1) goto Resume2;

            awaiter1 = repository.FindAsync(id, ct).GetAwaiter();
            if (!awaiter1.IsCompleted)
            {
                state = 0;
                builder.AwaitUnsafeOnCompleted(ref awaiter1, ref this);   // "call MoveNext again when done"
                return;                                                    // give the thread back
            }
        Resume1:
            ticket = awaiter1.GetResult();                                 // result, or rethrows exception

            awaiter2 = users.GetAsync(ticket!.Assignee!, ct).GetAwaiter();
            if (!awaiter2.IsCompleted)
            {
                state = 1;
                builder.AwaitUnsafeOnCompleted(ref awaiter2, ref this);
                return;
            }
        Resume2:
            var user = awaiter2.GetResult();
            builder.SetResult($"{ticket.Title} — {user.DisplayName}");
        }
        catch (Exception ex)
        {
            builder.SetException(ex);                                      // exception goes into the Task
        }
    }
}
```

Read that carefully, because nearly every async rule follows from it:

1. **The method runs synchronously until the first incomplete `await`.** If `FindAsync`
   completes immediately (a cache hit, say), there's no suspension at all: the "fast path."
2. **At an incomplete await, the method *returns*** to its caller, handing back an
   incomplete `Task`. The thread is free.
3. **When the awaited task completes, `MoveNext` is called again** and jumps to the
   resume point. It may run on a *different thread*.
4. **Exceptions are caught and stored in the returned `Task`.** They're rethrown when
   someone awaits that task (`GetResult()`).
5. **Locals that cross an `await` become fields.** The state machine starts as a struct
   on the stack; on the first real suspension, it's *boxed* onto the heap so it can
   survive the method returning.

Paste any async method into SharpLab to see the real generated code; it's longer, but the
shape is exactly this.

---

## 3. Where does the code resume? `SynchronizationContext`

When an awaited task completes, *where* does the rest of the method run?

By default, `await` captures the current **`SynchronizationContext`** (or, if there is
none, the current `TaskScheduler`) and resumes there:

| Environment | Context | Resumes on |
|---|---|---|
| WinForms / WPF / MAUI UI thread | UI context | The UI thread (so you can update controls) |
| Classic ASP.NET (.NET Framework) | Request context | One thread at a time for the request |
| **ASP.NET Core** | **None** | **Any thread-pool thread** |
| Console apps, worker services | None | Any thread-pool thread |

### `ConfigureAwait(false)`

`await task.ConfigureAwait(false)` says "don't capture the context; resume on any
thread." It matters in two situations:

- **Library code** that might be called from a UI app: it avoids hopping back to the UI
  thread for no reason, and avoids the classic deadlock below.
- **Old ASP.NET** on .NET Framework.

In **ASP.NET Core application code**, there is no context, so `ConfigureAwait(false)`
does nothing useful. Most teams omit it in application code and use it in reusable
libraries.

### The classic deadlock

```csharp
// In a WinForms button handler, or old ASP.NET:
var result = GetDataAsync().Result;   // blocks the UI thread waiting for the task

async Task<string> GetDataAsync()
{
    await Task.Delay(100);            // captures the UI context
    return "done";                    // needs the UI thread to resume... which is blocked
}
```

The UI thread is blocked waiting for the task; the task is waiting for the UI thread to
be free so it can resume. Neither can proceed. It's a deadlock, and it happens only when
there's a single-threaded context, which is why it "works on my machine" in a console app
and hangs in the desktop app.

The real fix is not `ConfigureAwait(false)`; it's **don't block on async code**:
"async all the way."

---

## 4. The rules, and why

### Rule 1: Async all the way. Don't block on tasks.

`.Result`, `.Wait()` and `.GetAwaiter().GetResult()` block the current thread until the
task completes. That causes:

- **Deadlocks** in contexts with a `SynchronizationContext` (section 3).
- **Thread-pool starvation** in ASP.NET Core (section 6): each blocked thread is one
  less thread to run the continuations that would unblock it.

If a method calls async code, make it async too, up to the entry point. `Main` can be
`async Task Main`; ASP.NET Core endpoints and controllers support async natively.

### Rule 2: Avoid `async void`

An `async void` method returns nothing, so:

- the caller can't await it or know when it's done,
- exceptions can't be stored in a task, so they're rethrown on the thread pool (or
  context) and **crash the process**.

Use `async Task`. The only legitimate `async void` is an event handler (`button.Click +=
async (s, e) => ...`), because the event signature demands `void`; wrap its body in
`try/catch`.

Watch for accidental `async void`: passing an async lambda to a parameter of type
`Action` (for example `list.ForEach(async x => await ...)`) creates one.

### Rule 3: Pass `CancellationToken`s through

Every async API should accept a `CancellationToken` and pass it to everything it calls.
In ASP.NET Core, `HttpContext.RequestAborted` is cancelled when the client disconnects;
flowing it means you stop querying the database for a user who's gone.

```csharp
app.MapGet("/tickets/{id}", async (int id, ITicketRepository repo, CancellationToken ct) =>
    await repo.FindAsync(new TicketId(id), ct));   // ct is bound to RequestAborted
```

### Rule 4: Don't use `Task.Run` to fake async in server code

`Task.Run(() => SyncWork())` moves work to another thread-pool thread. In a UI app,
that's useful (it keeps the UI responsive). In ASP.NET Core, it just swaps one thread-pool
thread for another, adding overhead and fooling readers into thinking the code is async.
For CPU-bound work in a server, run it synchronously; for I/O, use truly async APIs.

### Rule 5: Start independent operations together

```csharp
// Sequential: ~100 ms (20 + 80)
var ticket = await repository.FindAsync(id, ct);
var stats  = await statsApi.GetAsync(ct);

// Concurrent: ~80 ms (the longer of the two)
var ticketTask = repository.FindAsync(id, ct);
var statsTask  = statsApi.GetAsync(ct);
await Task.WhenAll(ticketTask, statsTask);
var ticket2 = ticketTask.Result;   // safe: already completed
var stats2  = statsTask.Result;
```

Only when the operations are genuinely independent, and only if the underlying resource
allows concurrency. A single EF Core `DbContext` does *not* allow concurrent queries.

---

## 5. `Task` vs `ValueTask`, and other variations

### `ValueTask<T>`

Every async method returning `Task<T>` allocates a `Task` object, unless the runtime can
reuse a cached one. For methods that **usually complete synchronously** (a cache that
hits 99% of the time), that's an allocation on every call for no benefit.

`ValueTask<T>` is a struct that can hold either a result directly (no allocation) or a
`Task<T>` (or a pooled object) for the asynchronous case.

```csharp
public ValueTask<Ticket?> FindAsync(TicketId id, CancellationToken ct)
{
    if (_cache.TryGetValue(id, out var cached))
        return ValueTask.FromResult<Ticket?>(cached);      // no allocation
    return new ValueTask<Ticket?>(LoadAsync(id, ct));       // falls back to a Task
}
```

The catch: a `ValueTask` must be **awaited exactly once**. You can't await it twice, await
it concurrently, or call `.Result` before it completes. (Some implementations reuse the
underlying object after the first await.)

> **🧭 When not to use it:** Default to `Task<T>`. Use `ValueTask<T>` for hot-path methods
> that frequently complete synchronously and when a profiler shows the allocations
> matter. It's common in the BCL (`Stream.ReadAsync`, `Channel` readers); it's rarely
> needed in application code.

### Async streams: `IAsyncEnumerable<T>`

For sequences whose elements arrive asynchronously (database rows, paged API results,
messages):

```csharp
public async IAsyncEnumerable<Ticket> StreamOpenAsync([EnumeratorCancellation] CancellationToken ct = default)
{
    int page = 0;
    while (true)
    {
        var batch = await LoadPageAsync(page++, ct);
        if (batch.Count == 0) yield break;
        foreach (var t in batch) yield return t;
    }
}

await foreach (var t in StreamOpenAsync(ct))
    Console.WriteLine(t.Title);
```

It's `yield return` plus `await`: the compiler generates a state machine combining both.

### Useful `Task` combinators

- `Task.WhenAll(tasks)`: completes when all complete; the awaited exception is the first
  failure (inspect `task.Exception` for all).
- `Task.WhenAny(tasks)`: completes when any completes. Useful for timeouts and racing.
- `Task.WhenEach(tasks)` (.NET 9): an `IAsyncEnumerable` yielding tasks as they complete.
- `task.WaitAsync(timeout, ct)`: await with a timeout or cancellation (.NET 6).
- `Task.Delay(time, ct)`: an async, cancellable sleep. Never use `Thread.Sleep` in async code.

---

## 6. Thread-pool starvation: the production failure

This failure deserves its own section because it's so common and so confusing.

**Symptoms:** under load, an ASP.NET Core service's latency climbs to seconds, requests
time out, but CPU is low and the database looks fine. Then it may recover by itself.

**Cause:** synchronous blocking on async work (`.Result`, `.Wait()`, synchronous I/O APIs)
inside request handling. Each blocked request holds a thread-pool thread. When all
threads are blocked, the continuations that would *unblock* them have no thread to run on.
The pool injects new threads only slowly (roughly one or two per second once past its
minimum), so the service crawls.

> **🔍 Investigation: suspected thread-pool starvation.**
> 1. Check `dotnet-counters monitor -n Beacon.Api System.Runtime` and watch
>    **ThreadPool Thread Count** (climbing steadily) and **ThreadPool Queue Length**
>    (growing).
> 2. Capture stacks with `dotnet-stack report -p <pid>` or a dump (`dotnet-dump collect`),
>    and look for many threads parked in `Task.Wait`, `TaskAwaiter.GetResult` or
>    `Monitor.Wait` called from your code.
> 3. Fix the sync-over-async call sites. Raising `ThreadPool.SetMinThreads` hides the
>    symptom; it doesn't fix the cause.

---

## 7. In practice: an async repository for Beacon

Chapter 8 introduced `ITicketRepository` with async methods. Let's implement an in-memory
version properly, and use the patterns from this chapter in the CLI.

```csharp
// src/Beacon.Cli/InMemoryTicketRepository.cs
using System.Collections.Concurrent;
using Beacon.Core.Tickets;

internal sealed class InMemoryTicketRepository : ITicketRepository
{
    private readonly ConcurrentDictionary<TicketId, Ticket> _tickets = new();
    private readonly TimeSpan _simulatedLatency;

    public InMemoryTicketRepository(TimeSpan simulatedLatency) => _simulatedLatency = simulatedLatency;

    public async Task<Ticket?> FindAsync(TicketId id, CancellationToken ct = default)
    {
        await Task.Delay(_simulatedLatency, ct);              // pretend to be a database
        return _tickets.GetValueOrDefault(id);
    }

    public async Task SaveAsync(Ticket ticket, CancellationToken ct = default)
    {
        await Task.Delay(_simulatedLatency, ct);
        _tickets[ticket.Id] = ticket;
    }
}
```

The CLI's entry point becomes properly async, wires cancellation to Ctrl+C, and loads
several tickets concurrently:

```csharp
// src/Beacon.Cli/Program.cs (excerpt)
using var cts = new CancellationTokenSource();
Console.CancelKeyPress += (_, e) => { e.Cancel = true; cts.Cancel(); };
var ct = cts.Token;

var repo = new InMemoryTicketRepository(TimeSpan.FromMilliseconds(200));
var now = TimeProvider.System.GetUtcNow();
foreach (var i in Enumerable.Range(1, 5))
    await repo.SaveAsync(new Ticket(new TicketId(i), $"Sample ticket {i}", TicketPriority.Normal, now), ct);

var sw = System.Diagnostics.Stopwatch.StartNew();
var ids = Enumerable.Range(1, 5).Select(i => new TicketId(i));

Ticket?[] found = await Task.WhenAll(ids.Select(id => repo.FindAsync(id, ct)));

Console.WriteLine($"Loaded {found.Count(t => t is not null)} tickets in {sw.ElapsedMilliseconds} ms");
// ≈ 200 ms, not 1,000 ms: the five lookups waited concurrently, without five threads.
```

Try replacing `Task.WhenAll` with a `foreach` and `await` inside it: about 1,000 ms. Then
try `.Result` inside a `Select` with a constrained thread pool to see starvation for
yourself (exercise 3).

---

## 8. What can go wrong

- **Sync over async** (`.Result`, `.Wait()`): deadlocks in UI/legacy contexts; thread-pool
  starvation in servers.
- **`async void`** crashing the process.
- **Fire-and-forget** (`_ = SendEmailAsync();`): exceptions are lost, and in ASP.NET Core
  the work may be cut off when the request ends or the app shuts down. Use a background
  queue (Book III, Chapter 8).
- **Forgetting to await**: the compiler warns (CS4014); don't suppress it. The method
  continues before the work is done, and exceptions are lost.
- **Not flowing `CancellationToken`**: wasted work for abandoned requests; slow shutdowns.
- **Concurrent use of non-thread-safe resources**, such as one `DbContext` in a
  `Task.WhenAll`.
- **Awaiting a `ValueTask` twice.**
- **Long CPU-bound work in async methods** blocking a thread-pool thread for seconds.
  Offload to a dedicated background worker or queue instead.

---

## 9. How an experienced engineer thinks about this

- **I/O-bound → async; CPU-bound → synchronous (or parallel, Chapter 10).** Async is for
  waiting, not computing.
- **Async all the way, from the entry point down.** No blocking calls in the middle.
- **Cancellation is part of every async signature.**
- **Concurrency is a decision.** `Task.WhenAll` when operations are independent and the
  resources allow it; sequential when they aren't.
- **When latency is high but CPU is low, suspect blocking.** Go straight to thread-pool
  counters and stack dumps.

---

## 10. Check yourself

**Questions**

1. Why is async about scalability rather than speed?
2. Is there a thread waiting while an async database query is in progress? Explain.
3. What does the compiler generate for an `async` method? What happens at an incomplete
   `await`?
4. Explain the classic `.Result` deadlock. Why doesn't it happen in ASP.NET Core?
5. Why is `async void` dangerous? When is it acceptable?
6. When is `ValueTask<T>` appropriate, and what's its main restriction?
7. Describe the symptoms and diagnosis of thread-pool starvation.

**Exercises**

1. Paste a two-`await` method into SharpLab and find the `state` field, the awaiter fields
   and the `MoveNext` method.
2. Measure sequential vs `Task.WhenAll` loading in the Beacon CLI.
3. In a console app, call `ThreadPool.SetMaxThreads(4, 4)`, then run 20 tasks that each
   block on `Task.Delay(1000).Wait()`. Time it. Then rewrite with `await` and compare.
4. Implement `TicketService.FindManyAsync(IEnumerable<TicketId>)` that loads tickets with
   at most 3 concurrent lookups. (Hint: `Parallel.ForEachAsync` with `MaxDegreeOfParallelism`,
   or `SemaphoreSlim`.)

**Interview-style questions**

- "What happens when you `await` a task that hasn't completed?"
- "What is `ConfigureAwait(false)` and when should you use it?"
- "Why shouldn't you call `.Result` on a task?"
- "Our API is slow under load but the CPU is idle. What would you check?"

---

## 11. Going deeper

- Stephen Toub, [How async/await really works in C#](https://devblogs.microsoft.com/dotnet/how-async-await-really-works/)
  — the definitive long-form explanation, from the .NET team.
- Stephen Cleary, [There Is No Thread](https://blog.stephencleary.com/2013/11/there-is-no-thread.html)
  and his book *Concurrency in C# Cookbook*.
- David Fowler's [Async Guidance](https://github.com/davidfowl/AspNetCoreDiagnosticScenarios/blob/master/AsyncGuidance.md)
  for ASP.NET Core.

**Next:** [Chapter 10 — Threading and Concurrency](10-threading-and-concurrency.md)
covers what happens when multiple threads really do run at the same time, and how to
keep shared state correct.
