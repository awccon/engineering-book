# Memory, the GC and Performance

The garbage collector is the runtime service you depend on most and think about least.
That's mostly good: it frees you from manual memory management. But when an API's p99
latency spikes every few seconds, when a container is killed for exceeding its memory
limit, or when memory climbs for days until a restart, you need to know how the GC works,
what it costs, and how to measure what's really happening.

This chapter also covers the discipline of performance work in general: measure, find
the real bottleneck, change one thing, measure again.

---

## 1. The problem: memory management is a trade-off

Every program allocates memory and must eventually release it. There are three main
approaches:

| Approach | Used by | Benefit | Cost |
|---|---|---|---|
| Manual (`malloc`/`free`) | C, C++ (partially) | Full control, predictable | Leaks, use-after-free, double-free |
| Ownership / borrowing | Rust | Safe *and* no runtime cost | Learning curve, compiler restrictions (Book XII) |
| Garbage collection | .NET, Java, Go, JavaScript | Safe and easy | Run-time cost: CPU, pauses, extra memory |

.NET chose garbage collection. The question isn't whether it costs something (it does),
but how to keep that cost low.

---

## 2. The mental model: how the .NET GC works

### Allocation is cheap

The managed heap allocates by **bumping a pointer**: the runtime keeps a pointer to the
next free byte in the current allocation region; `new` just advances it. This is far
cheaper than a typical `malloc`, which must search for free space.

### Collection: find what's alive, discard the rest

The GC doesn't track dead objects. It finds **live** ones:

1. Start from **roots**: local variables and parameters on every thread's stack, static
   fields, CPU registers, GC handles.
2. **Mark** every object reachable from a root, following references.
3. Everything not marked is garbage. **Sweep** it or **compact** the heap by sliding live
   objects together and updating references.

```text
  Roots                Heap
  ┌──────────┐         ┌────┐   ┌────┐   ┌────┐   ┌────┐   ┌────┐
  │ local t ─┼────────►│ A ─┼──►│ B  │   │ C  │   │ D ─┼──►│ E  │
  │ static s─┼─────────┼────┼───┼────┼──►│    │   │    │   │    │
  └──────────┘         └────┘   └────┘   └────┘   └────┘   └────┘
                        live     live     live    garbage  garbage (D→E but nothing→D)
```

The cost of a collection is proportional to the number of **live** objects (what must be
marked and moved), not dead ones. Dead objects are free to collect.

### Generations

The GC exploits an observation called the **generational hypothesis**: *most objects die
young.* A request's DTOs, strings and LINQ iterators live for milliseconds. Caches and
singletons live for the app's lifetime.

So the heap is divided into generations:

| Generation | Contains | Collected | Typical cost |
|---|---|---|---|
| **Gen 0** | Newly allocated objects | Very often | Very cheap (small, mostly dead) |
| **Gen 1** | Survived one gen0 collection | Less often | A buffer between 0 and 2 |
| **Gen 2** | Long-lived objects | Rarely | Expensive (large, mostly alive) |
| **LOH** (Large Object Heap) | Objects ≥ 85,000 bytes | With gen 2 | Not compacted by default |
| **POH** (Pinned Object Heap) | Objects allocated as pinned | With gen 2 | Never moved |

Objects that survive a collection are **promoted** to the next generation. A gen0
collection only looks at gen0 objects, which is fast because almost all of them are dead.

> **🧱 Durable:** Short-lived objects are cheap. Long-lived objects are cheap. Expensive
> objects are the ones that live **just long enough** to be promoted to gen 2 and then
> die: a cache that holds things for a few minutes, buffers held across slow I/O. They
> drive expensive full collections.

### Pauses: workstation vs server GC, background GC

During parts of a collection, the GC must pause your threads so references don't change
underneath it.

- **Workstation GC**: one heap, tuned for desktop responsiveness. The default for console
  and desktop apps.
- **Server GC**: one heap and GC thread **per core**, tuned for throughput. The default
  for ASP.NET Core. Uses more memory.
- **Background GC**: gen2 collections run mostly concurrently with your code; only short
  pauses are needed.
- **DATAS** (Dynamic Adaptation To Application Sizes): makes Server GC grow and shrink its
  heap count based on load, rather than always using one heap per core.

> **🔄 Current (as of October 2026):** DATAS is enabled by default with Server GC since
> .NET 9, which substantially reduced the memory footprint of ASP.NET Core apps in
> containers. GC settings are configured in the project file or `runtimeconfig.json`
> (`<ServerGarbageCollector>`, `<ConcurrentGarbageCollection>`) or with `DOTNET_gc*`
> environment variables.

### The Large Object Heap

Objects of 85,000 bytes or more (large arrays, big strings, large `MemoryStream` buffers)
go straight to the LOH, are collected only with gen 2, and by default aren't compacted.
Repeatedly allocating large temporary buffers causes gen2 collections and fragmentation.
The standard fix is **pooling** (section 5).

---

## 3. "Managed" doesn't mean "can't leak"

The GC collects unreachable objects. A **managed memory leak** is an object that's no
longer *needed* but is still *reachable*. Common sources:

- **Static collections** that only grow (`static List<RequestLog>`).
- **Caches without expiration or size limits.**
- **Event subscriptions** from long-lived publishers to short-lived subscribers
  (Chapter 6).
- **Captured variables** in long-lived delegates (timers, callbacks).
- **Captive dependencies** in DI: a singleton holding a scoped object (Chapter 13).

### `IDisposable`: resources, not memory

The GC manages **memory**. It doesn't promptly release **other resources**: file handles,
sockets, database connections, OS handles, native memory. Types that hold them implement
`IDisposable`, and you must dispose them, usually with `using`:

```csharp
await using var stream = File.OpenRead(path);     // closed at end of scope
using var timer = new PeriodicTimer(TimeSpan.FromSeconds(5));
```

**Finalizers** (`~MyType()`) exist as a safety net for unmanaged resources, but they run
at an unpredictable time on a separate thread, and objects with finalizers survive an
extra collection. You'll almost never write one; wrap native handles in `SafeHandle`
instead.

> **⚠️ What can go wrong:** `HttpClient` is the classic disposal trap in reverse. Creating
> and disposing an `HttpClient` per request exhausts sockets, because closed connections
> linger in the `TIME_WAIT` state. Use `IHttpClientFactory` or a long-lived shared client
> (Book III).

---

## 4. Measuring before optimizing

Performance work without measurement is guessing. The workflow:

1. **Define the goal.** "p99 latency under 200 ms at 500 requests/s," not "faster."
2. **Measure the whole system** in a realistic environment (load test, production metrics).
3. **Find the bottleneck.** It's usually the database, the network, or one hot loop.
4. **Change one thing**, measure again, keep or revert.

### Tools

| Tool | What it tells you |
|---|---|
| `dotnet-counters` | Live metrics: GC counts per generation, heap size, allocation rate, % time in GC, thread pool, exceptions |
| `dotnet-trace` | CPU sampling and runtime events; open in PerfView, Visual Studio or speedscope |
| `dotnet-gcdump` | Snapshot of the managed heap: which types use memory, and what holds them |
| `dotnet-dump` | Full process dump for offline analysis (threads, heap, exceptions) |
| Visual Studio / Rider profilers | CPU, allocations and memory with a GUI |
| **BenchmarkDotNet** | Accurate microbenchmarks of individual methods |

```bash
dotnet tool install -g dotnet-counters
dotnet-counters monitor -n Beacon.Api --counters System.Runtime
```

Key counters to watch: **Allocation Rate**, **% Time in GC**, **Gen 0/1/2 GC Count**,
**GC Heap Size**, **Working Set**.

### BenchmarkDotNet

Microbenchmarks are hard to get right: JIT tiering, warm-up, CPU frequency scaling and
dead-code elimination all distort naive `Stopwatch` timings. BenchmarkDotNet handles them:

```csharp
// benchmarks/Beacon.Benchmarks/ParsingBenchmarks.cs
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]                         // reports allocations per operation
public class ParsingBenchmarks
{
    private const string Line = "1042,High,Cannot log in to the VPN";

    [Benchmark(Baseline = true)]
    public int SplitBased()
    {
        var parts = Line.Split(',');
        return int.Parse(parts[0]) + parts[2].Length;
    }

    [Benchmark]
    public int SpanBased()
    {
        ReadOnlySpan<char> line = Line;
        int first = line.IndexOf(',');
        int second = line[(first + 1)..].IndexOf(',') + first + 1;
        return int.Parse(line[..first]) + line[(second + 1)..].Length;
    }
}

public static class Program
{
    public static void Main(string[] args) => BenchmarkRunner.Run<ParsingBenchmarks>();
}
```

```bash
dotnet run -c Release --project benchmarks/Beacon.Benchmarks
```

The output gives mean time, error, standard deviation, ratio to baseline and **allocated
bytes per operation**. Expect the span version to be several times faster and allocate
zero bytes, while `Split` allocates an array and three strings per call. Run it yourself;
the exact numbers depend on your machine, and seeing them is the point.

> **🧭 When not to optimize:** If an operation runs once per request and takes 2 µs, making
> it take 0.5 µs is irrelevant next to a 20 ms database call. Optimize what the profiler
> says is hot. Clear code that's fast enough beats clever code that's slightly faster.

---

## 5. Reducing allocations (when it matters)

Once a profiler shows GC pressure in a hot path, the standard techniques:

- **Avoid needless intermediate objects**: spans instead of substrings and `Split`
  (Chapter 5), `StringBuilder` or interpolated string handlers instead of concatenation in
  loops.
- **Avoid boxing** and capturing lambdas in hot paths (Chapters 2 and 6).
- **Pre-size collections**: `new List<T>(capacity)`, `new Dictionary<K,V>(capacity)`.
- **Pool large buffers** with `ArrayPool<T>`:

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(64 * 1024);   // may return a larger array
try
{
    int read = await stream.ReadAsync(buffer.AsMemory(0, 64 * 1024), ct);
    Process(buffer.AsSpan(0, read));
}
finally
{
    ArrayPool<byte>.Shared.Return(buffer);                 // never use buffer after this
}
```

- **Pool expensive objects** with `ObjectPool<T>` (`Microsoft.Extensions.ObjectPool`).
- **`stackalloc`** small, fixed-size temporary buffers on the stack:
  `Span<byte> tmp = stackalloc byte[256];`
- **Use `struct`s** for small, short-lived values in large numbers (Chapter 2).
- **Use `ValueTask`** for frequently synchronous async methods (Chapter 9).

Each of these trades simplicity for speed. Apply them where measurements justify it.

---

## 6. Performance beyond memory

Allocation is only one dimension. In typical business applications, ranked by how often
they're the real culprit:

1. **Database access**: missing indexes, N+1 queries, fetching too much (Book IV).
2. **Network calls**: chatty APIs, sequential calls that could be concurrent, no caching.
3. **Algorithmic complexity**: O(n²) hidden in loops (Chapter 5).
4. **Blocking and contention**: sync-over-async, locks held too long (Chapters 9–10).
5. **Allocation and GC**: high allocation rates in hot paths.
6. **Raw CPU**: tight loops, which benefit from spans, SIMD and avoiding virtual calls.

Most teams should look in roughly that order.

---

## 7. In practice: investigating Beacon's memory

Let's simulate a realistic problem in the CLI and diagnose it with the tools.

### The bug

Someone adds an "audit log" to Beacon: every ticket operation is recorded so managers
can view recent activity.

```csharp
// src/Beacon.Core/Auditing/AuditLog.cs  (first version: leaks)
namespace Beacon.Core.Auditing;

public sealed record AuditEntry(DateTimeOffset At, string Actor, string Action, string Details);

public static class AuditLog
{
    private static readonly List<AuditEntry> Entries = new();   // grows forever, not thread-safe
    public static void Record(AuditEntry e) => Entries.Add(e);
    public static IEnumerable<AuditEntry> Recent(int n) => Entries.TakeLast(n);
}
```

Two problems: a static list that grows forever (a leak) and a `List<T>` mutated from
multiple threads (a race, Chapter 10).

### Diagnosing it

Simulate load in the CLI by recording a million entries with a 1 KB `Details` string
each, then:

```bash
dotnet-counters monitor -n Beacon.Cli --counters System.Runtime
```

You'll see **GC Heap Size** climbing steadily, **Gen 2 GC Count** rising, and heap size
never coming back down after gen2 collections. That pattern, memory that survives full
collections and keeps growing, means *something is holding references*.

```bash
dotnet-gcdump collect -n Beacon.Cli -o beacon.gcdump
dotnet-gcdump report beacon.gcdump | head -20
```

The report lists types by total size. `AuditEntry` and `System.String` will dominate.
Opening the dump in Visual Studio or PerfView shows the **path to root**:
`static AuditLog.Entries → List<AuditEntry> → AuditEntry[] → AuditEntry`.

### The fix

A bounded, thread-safe buffer that keeps only the most recent entries:

```csharp
// src/Beacon.Core/Auditing/AuditLog.cs  (fixed)
namespace Beacon.Core.Auditing;

public sealed record AuditEntry(DateTimeOffset At, string Actor, string Action, string Details);

public sealed class AuditLog(int capacity = 10_000)
{
    private readonly Queue<AuditEntry> _entries = new(capacity);
    private readonly Lock _gate = new();

    public void Record(AuditEntry entry)
    {
        lock (_gate)
        {
            if (_entries.Count == capacity) _entries.Dequeue();   // drop the oldest
            _entries.Enqueue(entry);
        }
    }

    public IReadOnlyList<AuditEntry> Recent(int n)
    {
        lock (_gate)
        {
            return _entries.Skip(Math.Max(0, _entries.Count - n)).ToArray();   // copy inside the lock
        }
    }
}
```

Changes: an instance (registered as a singleton in Book III, not a static), a fixed
capacity, a lock around every access, and `Recent` returns a **copy** so callers never
iterate the queue while another thread modifies it. The *durable* audit log will live in
PostgreSQL (Book IV); in memory we keep only a recent window.

Run the counters again: heap size plateaus.

---

## 8. What can go wrong

- **Leaks through reachability**: statics, unbounded caches, events, captive dependencies.
- **Mid-life crisis objects**: data held just long enough to reach gen 2, causing frequent
  full collections.
- **LOH churn**: repeatedly allocating large temporary buffers.
- **Undisposed resources**: socket exhaustion, file locks, connection-pool exhaustion
  ("Timeout expired... all pooled connections were in use").
- **Container memory limits**: the GC respects container limits, but a heap sized for
  the host machine, or native memory the GC doesn't count, can still get a container
  OOM-killed. Watch working set, not just heap size.
- **Optimizing the wrong thing**: shaving microseconds while a missing index costs seconds.
- **Benchmarking wrong**: Debug builds, no warm-up, tiny sample sizes, measuring dead code.

---

## 9. How an experienced engineer thinks about this

- **Measure, don't guess.** Counters first, then traces or dumps.
- **Think in object lifetimes.** "How long does this live, and what keeps it alive?" is
  the core memory question.
- **Bound everything that grows.** Caches, queues, buffers and logs need limits.
- **Distinguish memory from resources.** GC handles one; `Dispose` handles the other.
- **Optimize in proportion.** Clarity by default; allocation-conscious code in measured
  hot paths.

---

## 10. Check yourself

**Questions**

1. Why is allocation on the managed heap cheap?
2. What are GC roots? What determines the cost of a collection?
3. Explain the generational hypothesis and why gen0 collections are fast.
4. What's the Large Object Heap, and why does it matter?
5. How can a garbage-collected application leak memory? Give three examples.
6. Why does `IDisposable` exist if .NET has a garbage collector?
7. Why use BenchmarkDotNet instead of `Stopwatch`?

**Exercises**

1. Run the `ParsingBenchmarks` and explain the allocation column.
2. Reproduce the audit-log leak with `dotnet-counters` running. Capture a gcdump and find
   the path to root.
3. Benchmark `string.Join` vs `StringBuilder` vs `+=` for building a string from 10, 100
   and 10,000 parts.
4. Write a method that rents a buffer from `ArrayPool`, and deliberately use it after
   returning it. Why is this bug dangerous and hard to detect?

**Interview-style questions**

- "How does garbage collection work in .NET?"
- "Can you have a memory leak in .NET? How would you find one?"
- "What's the difference between `Dispose` and a finalizer?"
- "How would you approach a performance problem in a production API?"

---

## 11. Going deeper

- [Microsoft docs: Fundamentals of garbage collection](https://learn.microsoft.com/dotnet/standard/garbage-collection/fundamentals)
- [Microsoft docs: .NET diagnostic tools](https://learn.microsoft.com/dotnet/core/diagnostics/)
- Konrad Kokosa, *Pro .NET Memory Management*.
- [BenchmarkDotNet documentation](https://benchmarkdotnet.org/)
- Stephen Toub's yearly *Performance Improvements in .NET* posts.

**Next:** [Chapter 12 — Reflection, Attributes and Source Generators](12-reflection-attributes-and-source-generators.md)
covers how code can inspect and generate code, and why the modern answer is increasingly
"at compile time."
