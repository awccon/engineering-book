# Threading and Concurrency

Chapter 9 was about *waiting* efficiently. This chapter is about what happens when
several pieces of code genuinely run **at the same time** and touch the same data.
Concurrency bugs are the hardest bugs in software: they depend on timing, so they appear
under production load, vanish when you add logging, and can't be reproduced on demand.
The defense isn't cleverness; it's a small set of well-understood tools and the
discipline to avoid shared mutable state wherever possible.

---

## 1. The problem: shared state and interleaving

Here's the smallest concurrency bug:

```csharp
int count = 0;
Parallel.For(0, 100_000, _ => count++);
Console.WriteLine(count);   // usually less than 100,000, and different every run
```

`count++` looks like one operation, but it's three: **read** `count`, **add** 1, **write**
it back. Two threads can interleave:

```text
 Thread A              Thread B              count
 read count (41)                               41
                       read count (41)         41
 write 42                                      42
                       write 42                42   ← one increment lost
```

This is a **race condition**: the result depends on the timing of threads. It happens to
any read-modify-write on shared data: counters, `Dictionary` inserts, "check then act"
(`if (!cache.ContainsKey(k)) cache.Add(k, v)`), lazy initialization.

Modern servers are concurrent by default: ASP.NET Core runs many requests in parallel on
many threads. Any **singleton** service or **static** field is shared by all of them.

---

## 2. The mental model

### Concurrency vs parallelism

- **Concurrency**: multiple tasks *in progress* at overlapping times (they may take turns
  on one core). Async I/O is concurrency.
- **Parallelism**: multiple tasks *executing* simultaneously on multiple cores. Processing
  an image on 8 cores is parallelism.

### Threads and the thread pool

A **thread** is an OS-scheduled sequence of execution with its own stack. Creating one is
expensive (an OS call, a stack allocation). So .NET maintains a **thread pool**: a set of
worker threads that execute queued work items. `Task.Run`, async continuations, timers,
and ASP.NET Core request processing all use it.

You almost never create `new Thread(...)` directly. The exceptions are long-running
dedicated loops (and even then `Task.Factory.StartNew(..., TaskCreationOptions.LongRunning)`
or a hosted `BackgroundService` is usually better).

### The three hazards

1. **Atomicity**: a multi-step operation can be interrupted midway (the `count++` problem).
2. **Visibility**: a write on one thread may not be *seen* by another thread promptly,
   because of CPU caches and compiler/JIT reordering.
3. **Ordering**: operations can be reordered by the compiler, JIT or CPU, as long as a
   single thread can't tell the difference. Another thread *can* tell.

The **memory model** defines what's guaranteed. You don't need its details if you follow
one rule: **all access to shared mutable state goes through a synchronization primitive**
(a lock, an `Interlocked` operation, a concurrent collection, or a `volatile` field used
correctly). Every primitive provides atomicity, visibility and ordering guarantees.

> **🧱 Durable:** The hierarchy of solutions, from best to worst: **don't share** (each
> thread has its own data) → **don't mutate** (share immutable data) → **use a
> purpose-built concurrent structure** → **lock** → **lock-free tricks**. Most bugs come
> from jumping straight to the bottom of that list, or from not noticing that data is
> shared at all.

---

## 3. Locks

### `lock`

```csharp
private readonly Lock _gate = new();      // .NET 9+: System.Threading.Lock
private int _count;

public void Increment()
{
    lock (_gate)
    {
        _count++;                          // only one thread at a time runs this block
    }
}
```

A `lock` ensures **mutual exclusion**: only one thread can hold it at a time; others wait.
It also guarantees visibility: everything written inside the lock is visible to the next
thread that takes it.

> **🔄 Current (as of October 2026):** .NET 9 introduced `System.Threading.Lock`, a
> dedicated lock type. The `lock` statement recognizes it and uses its faster API.
> On older versions, lock on a private `readonly object`.

Rules:

- **Lock on a private object** you own. Never `lock (this)`, `lock (typeof(X))` or a
  string: outside code can lock on those too.
- **Keep the critical section short.** Never do I/O or call unknown code (events,
  callbacks, virtual methods) inside a lock.
- **You can't `await` inside a `lock`.** The compiler forbids it, because the continuation
  might run on a different thread, and locks are thread-affine. Use `SemaphoreSlim` for
  async code.

### `SemaphoreSlim`: async-friendly mutual exclusion and throttling

```csharp
private readonly SemaphoreSlim _mutex = new(1, 1);   // 1 = mutual exclusion

public async Task RefreshAsync(CancellationToken ct)
{
    await _mutex.WaitAsync(ct);
    try
    {
        await ReloadFromDatabaseAsync(ct);
    }
    finally
    {
        _mutex.Release();
    }
}
```

With a count greater than one, a semaphore **throttles**: "at most 5 concurrent calls to
the payment API."

### `ReaderWriterLockSlim`

Allows many concurrent readers *or* one writer. Useful when reads vastly outnumber writes
and the critical sections are long enough to matter. Often, though, an immutable snapshot
swapped atomically (section 5) is simpler and faster.

### Deadlock

Two threads, two locks, opposite order:

```text
 Thread A: lock(accounts) → waits for lock(audit)
 Thread B: lock(audit)    → waits for lock(accounts)       ⇒ both wait forever
```

Prevention: **always acquire multiple locks in the same global order**, avoid holding a
lock while calling code you don't control, and prefer designs that need only one lock.

---

## 4. `Interlocked` and `volatile`

For single-variable updates, `Interlocked` provides atomic operations implemented with
CPU instructions, no lock needed:

```csharp
Interlocked.Increment(ref _count);
Interlocked.Add(ref _total, amount);
var old = Interlocked.Exchange(ref _current, newValue);
Interlocked.CompareExchange(ref _state, newState, expectedState);   // set only if unchanged
```

`CompareExchange` (CAS) is the building block of lock-free algorithms: read, compute, and
write back *only if nobody changed it in between*; otherwise retry.

`volatile` (or `Volatile.Read`/`Write`) guarantees visibility and prevents certain
reorderings for a single field, but **not atomicity** of read-modify-write. A `volatile
bool _stopping` flag is a fine use; `volatile int` with `++` is still a race.

> **🧭 When not to use it:** Lock-free code is extremely hard to get right and to review.
> Use `Interlocked` for counters and simple flags; use locks or concurrent collections
> for anything more complex. Leave lock-free data structures to the BCL authors.

---

## 5. Concurrent collections and immutability

`System.Collections.Concurrent` provides thread-safe collections designed for concurrent
access:

| Type | Use |
|---|---|
| `ConcurrentDictionary<TKey, TValue>` | Shared caches and lookups |
| `ConcurrentQueue<T>` / `ConcurrentStack<T>` | Lock-free FIFO / LIFO |
| `ConcurrentBag<T>` | Unordered items, optimized for the same thread adding and taking |
| `BlockingCollection<T>` | Older producer/consumer (prefer `Channel<T>`) |

### `ConcurrentDictionary` pitfalls

Each individual operation is atomic, but *sequences* of operations aren't:

```csharp
if (!cache.ContainsKey(id))                  // ✗ check-then-act race
    cache[id] = await LoadAsync(id);

var value = cache.GetOrAdd(id, key => Load(key));   // ✓ one atomic operation...
```

...but note that `GetOrAdd`'s factory **may run more than once** if two threads race;
only one result is stored. If the factory is expensive or has side effects, store a
`Lazy<T>` instead:

```csharp
private readonly ConcurrentDictionary<TicketId, Lazy<Task<Ticket?>>> _cache = new();

public Task<Ticket?> GetAsync(TicketId id) =>
    _cache.GetOrAdd(id, key => new Lazy<Task<Ticket?>>(() => LoadAsync(key))).Value;
```

Now the load runs exactly once per key, and concurrent callers share the same task.

### Immutable snapshots

Often the simplest thread-safe design is: readers use an immutable snapshot; writers
build a new snapshot and swap the reference atomically.

```csharp
private volatile FrozenDictionary<string, TicketPriority> _rules = LoadRules();

public TicketPriority Lookup(string label) => _rules[label];        // readers: no lock
public void Reload() => _rules = LoadRules();                       // writer: atomic reference swap
```

Reference assignment is atomic in .NET, and readers either see the old or the new
dictionary, never a half-built one.

---

## 6. Channels: producer/consumer done right

Many concurrency problems are really **pipelines**: one part produces work, another
consumes it. `System.Threading.Channels` provides an async-friendly, high-performance
queue for exactly this:

```csharp
var channel = Channel.CreateBounded<Ticket>(new BoundedChannelOptions(capacity: 100)
{
    FullMode = BoundedChannelFullMode.Wait,     // producers wait when full: back-pressure
});

// Producer
await channel.Writer.WriteAsync(ticket, ct);
channel.Writer.Complete();                      // no more items

// Consumer
await foreach (var t in channel.Reader.ReadAllAsync(ct))
    await IndexAsync(t, ct);
```

Why channels are such a good default:

- **No shared mutable state** in your code: data is *passed* between stages, not shared.
- **Back-pressure**: a bounded channel slows producers down when consumers can't keep up,
  instead of growing memory without limit.
- **Async all the way**: waiting for space or items doesn't block threads.

> **🧱 Durable:** "Share memory by communicating, don't communicate by sharing memory."
> (from Go's design philosophy). Message passing between independent workers is easier to
> reason about than many threads touching shared objects. You'll see the same idea at
> larger scale in message queues (Book XIII).

---

## 7. Data parallelism

For CPU-bound work over many items, use the parallel APIs rather than managing threads:

```csharp
// CPU-bound: compute something for each item, on all cores
Parallel.ForEach(documents, doc => doc.Embedding = ComputeHash(doc.Text));

// Async I/O with bounded concurrency
await Parallel.ForEachAsync(ticketIds,
    new ParallelOptions { MaxDegreeOfParallelism = 8, CancellationToken = ct },
    async (id, token) => await reindexer.ReindexAsync(id, token));

// PLINQ
var results = items.AsParallel().Where(Expensive).Select(Transform).ToList();
```

Parallelism has overhead (partitioning, scheduling, merging). For small amounts of cheap
work, it's slower than a simple loop. And in a web server, using all cores for one
request steals them from other requests.

> **🧭 When not to use it:** In ASP.NET Core request handlers, avoid `Parallel.For` and
> PLINQ for CPU work. The server already parallelizes across requests. Use data
> parallelism in batch jobs, background workers and CLI tools.

---

## 8. Thread safety in ASP.NET Core services

The most practical form of this chapter: know which of your objects are shared.

| DI lifetime (Chapter 13) | Shared across requests? | Must be thread-safe? |
|---|---|---|
| Singleton | **Yes**, by every concurrent request | **Yes** |
| Scoped | No, one per request | Usually no (unless you start parallel work within a request) |
| Transient | No | No |

Common bugs:

- A singleton with a `List<T>` or `Dictionary<TKey, TValue>` field mutated per request.
- A singleton holding a scoped `DbContext` (a *captive dependency*): one context shared by
  all requests concurrently, which corrupts it.
- Static mutable fields used as "a quick cache."

---

## 9. In practice: concurrency in Beacon

Two improvements, both straight from this chapter.

### A safe in-memory ID generator

The in-memory repository needs IDs for new tickets. Multiple concurrent callers must
never get the same ID:

```csharp
// src/Beacon.Cli/TicketIdGenerator.cs
using Beacon.Core.Tickets;

internal sealed class TicketIdGenerator(int start = 0)
{
    private int _last = start;
    public TicketId Next() => new(Interlocked.Increment(ref _last));
}
```

### A background indexing pipeline

Later, Beacon will index tickets for search (Book XI). The pattern starts here: request
handling *enqueues* work quickly; a background consumer processes it.

```csharp
// src/Beacon.Core/Indexing/IndexingQueue.cs
using System.Threading.Channels;
using Beacon.Core.Tickets;

namespace Beacon.Core.Indexing;

public sealed class IndexingQueue
{
    private readonly Channel<TicketId> _channel = Channel.CreateBounded<TicketId>(
        new BoundedChannelOptions(1_000)
        {
            FullMode = BoundedChannelFullMode.Wait,
            SingleReader = true,       // lets the channel use a faster implementation
        });

    public ValueTask EnqueueAsync(TicketId id, CancellationToken ct = default)
        => _channel.Writer.WriteAsync(id, ct);

    public IAsyncEnumerable<TicketId> ReadAllAsync(CancellationToken ct)
        => _channel.Reader.ReadAllAsync(ct);

    public void Complete() => _channel.Writer.Complete();
}
```

```csharp
// src/Beacon.Cli/Program.cs (excerpt)
var queue = new IndexingQueue();

var consumer = Task.Run(async () =>
{
    await foreach (var id in queue.ReadAllAsync(ct))
    {
        await Task.Delay(50, ct);                 // pretend to index
        Console.WriteLine($"[indexer] indexed {id}");
    }
}, ct);

var ids = new TicketIdGenerator();
await Parallel.ForEachAsync(Enumerable.Range(1, 20), ct, async (_, token) =>
{
    var id = ids.Next();                          // safe from many threads
    await queue.EnqueueAsync(id, token);
});

queue.Complete();      // signal: no more work
await consumer;        // wait for the indexer to drain the queue
```

Note what isn't here: no `lock`, no shared list. Twenty concurrent producers hand IDs to
one consumer through the channel, and the only shared mutable state, the ID counter, is
updated atomically. In Book III, the consumer becomes a `BackgroundService` inside the API.

---

## 10. What can go wrong

- **Race conditions** on shared mutable state, especially in singletons and statics.
- **Check-then-act** sequences on concurrent collections.
- **Deadlocks** from inconsistent lock ordering or from calling unknown code while
  holding a lock.
- **Locks held across slow operations**, serializing the whole application.
- **`lock` with `await`**: forbidden by the compiler; using `Monitor` manually to get
  around it is a bug.
- **Unbounded producer/consumer queues** growing until the process runs out of memory.
- **Over-parallelization** in web servers, starving other requests.
- **Heisenbugs**: adding logging changes the timing and hides the bug. Reason about the
  code; don't rely on reproducing it.

---

## 11. How an experienced engineer thinks about this

- **First question: is this state shared?** Singletons, statics, captured variables in
  parallel loops, and caches are shared. Per-request objects usually aren't.
- **Design to avoid sharing.** Immutable data, message passing, per-request state.
- **Use the highest-level tool that fits.** Channels, concurrent collections and
  `Parallel.ForEachAsync` before locks; locks before `Interlocked` tricks.
- **Make thread-safety part of a type's documentation.** "This class is thread-safe" or
  "not thread-safe; use one per request" saves the next developer from guessing.
- **Concurrency bugs are found by review and reasoning more than by testing.** Treat
  shared mutable state as a review red flag.

---

## 12. Check yourself

**Questions**

1. Why is `count++` not thread-safe?
2. What's the difference between concurrency and parallelism?
3. Why can't you `await` inside a `lock`? What do you use instead?
4. What's wrong with `if (!dict.ContainsKey(k)) dict[k] = v;` on a `ConcurrentDictionary`?
5. Why might `GetOrAdd`'s factory run twice, and how do you prevent that?
6. What is back-pressure, and how do bounded channels provide it?
7. Which DI lifetime requires thread-safe services, and why?

**Exercises**

1. Fix the `Parallel.For` counter three ways: `lock`, `Interlocked`, and per-thread
   partial counts combined at the end (`Parallel.For` with thread-local state). Compare
   speeds.
2. Create a deliberate deadlock with two locks, observe it hang, then fix it with lock
   ordering.
3. Extend the indexing pipeline with three consumers instead of one. What needs to change?
   (Hint: `SingleReader`.)
4. Write a thread-safe cache using `ConcurrentDictionary<TKey, Lazy<Task<TValue>>>` and
   verify with a counter that each key loads exactly once under concurrent access.

**Interview-style questions**

- "What's a race condition? Give an example from a web application."
- "How would you implement a thread-safe cache?"
- "What's a deadlock, and how do you prevent one?"
- "What's the producer/consumer pattern and how would you implement it in .NET?"

---

## 13. Going deeper

- [Microsoft docs: Managed threading best practices](https://learn.microsoft.com/dotnet/standard/threading/managed-threading-best-practices)
- [Microsoft docs: System.Threading.Channels](https://learn.microsoft.com/dotnet/core/extensions/channels)
- Joe Albahari, *Threading in C#* (free online) — a thorough classic.
- Stephen Cleary, *Concurrency in C# Cookbook*.

**Next:** [Chapter 11 — Memory, the GC and Performance](11-memory-the-gc-and-performance.md)
looks at what the garbage collector does, what allocations cost, and how to measure
performance properly.
