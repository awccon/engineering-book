# Collections and Data Structures

Choosing a collection looks like a minor decision. It isn't. The wrong one turns an
O(1) lookup into an O(n) scan inside a loop, and a page that loaded in 50 ms with test
data takes 30 seconds with production data. The right one makes code faster *and* more
expressive: a `HashSet<T>` says "these are unique," a `Queue<T>` says "first in, first
out."

This chapter covers the data structures behind .NET's collections, how to reason about
their costs, the immutable and frozen collections, and `Span<T>` and `Memory<T>`, which
changed how high-performance .NET code is written.

---

## 1. The problem: performance that depends on data size

Here's a real-world bug in miniature:

```csharp
// For each of 50,000 tickets, check if its assignee is on the escalation list.
List<string> escalationList = LoadEscalationList();   // 5,000 names
var escalated = tickets.Where(t => escalationList.Contains(t.Assignee!)).ToList();
```

`List<T>.Contains` checks every element. That's up to 5,000 comparisons per ticket, or
250 million comparisons in total. Change one line:

```csharp
var escalationSet = new HashSet<string>(LoadEscalationList(), StringComparer.Ordinal);
```

Now each `Contains` is a hash lookup: roughly constant time. 50,000 lookups instead of
250 million comparisons. Same result, thousands of times faster.

That's the whole chapter in one example: **know the cost of each operation, and pick the
structure whose cheap operations match what your code does most.**

---

## 2. The mental model: Big-O in practice

**Big-O notation** describes how an operation's cost grows with the number of elements, n.

| Notation | Name | Example | 1,000 items | 1,000,000 items |
|---|---|---|---|---|
| O(1) | Constant | Array index, hash lookup | 1 | 1 |
| O(log n) | Logarithmic | Binary search, sorted tree lookup | ~10 | ~20 |
| O(n) | Linear | Scanning a list | 1,000 | 1,000,000 |
| O(n log n) | Linearithmic | Sorting | ~10,000 | ~20,000,000 |
| O(n²) | Quadratic | Nested loops over the same data | 1,000,000 | 10¹² |

Three practical refinements to the textbook version:

1. **Constants matter for small n.** Scanning an array of 8 items is usually faster than
   hashing. For tiny collections, simplicity wins.
2. **Memory layout matters a lot.** CPUs read memory in cache lines (64 bytes). Iterating
   a contiguous array is far faster than chasing pointers through a linked list, even
   when both are O(n). This is why `List<T>` beats `LinkedList<T>` for almost
   everything.
3. **"Amortized" means "on average."** Adding to a `List<T>` is usually O(1), but
   occasionally the list must grow its internal array, which costs O(n). Averaged over
   many adds, it's O(1) *amortized*.

> **🧱 Durable:** The most common performance bug in business software isn't a slow
> algorithm; it's an **O(n) operation hidden inside a loop**, making the whole thing
> O(n²). `List.Contains`, `List.Remove`, `IndexOf`, `.First(x => ...)` and LINQ
> `Where` inside a `foreach` are the usual suspects.

---

## 3. The core collections

### `T[]` (arrays)

A fixed-size, contiguous block of memory.

- Index: O(1). Iterate: O(n), and very cache-friendly.
- Fixed size. "Resizing" means allocating a new array and copying.
- Use for fixed-size data, buffers, and as the backing store of other collections.

### `List<T>`

A resizable array. Internally a `T[]` plus a `Count`. When full, it allocates a new array
twice the size and copies.

| Operation | Cost |
|---|---|
| `list[i]`, `list[i] = x` | O(1) |
| `Add` | O(1) amortized |
| `Insert(0, x)`, `RemoveAt(0)` | O(n): shifts every element |
| `Contains`, `IndexOf`, `Remove(x)` | O(n) |
| `Sort` | O(n log n) |

If you know the size in advance, pass a capacity: `new List<Ticket>(expectedCount)`
avoids repeated growth and copying.

### `Dictionary<TKey, TValue>`

A **hash table**. To store a key, it computes `key.GetHashCode()`, uses that to pick a
*bucket*, and stores the entry there. To find a key, it hashes again and looks in that
bucket, comparing with `Equals`.

```text
  key "maria" ──GetHashCode()──► 0x5A3F...  ──mod buckets──► bucket 3
                                                            ┌─────────┐
  buckets: [0] [1] [2] [3]──────────────────────────────────► "maria" → Ticket list
                                                            │ "omar"  → ... (collision)
                                                            └─────────┘
```

- Add, lookup, remove: O(1) on average.
- Depends entirely on **good `GetHashCode` and consistent `Equals`** (Chapter 2). A hash
  function that returns the same value for everything turns the dictionary into a list.
- Iteration order is not guaranteed. Don't depend on it.

### `HashSet<T>`

A dictionary with only keys. O(1) `Add`, `Contains`, `Remove`. Supports set operations:
`UnionWith`, `IntersectWith`, `ExceptWith`, `IsSubsetOf`.

### `Queue<T>`, `Stack<T>`, `PriorityQueue<TElement, TPriority>`

- `Queue<T>`: first in, first out. O(1) `Enqueue` / `Dequeue`. Good for work lists and
  breadth-first traversal.
- `Stack<T>`: last in, first out. O(1) `Push` / `Pop`. Good for undo, depth-first
  traversal, parsing.
- `PriorityQueue<TElement, TPriority>` (.NET 6+): a binary heap. O(log n) enqueue and
  dequeue, always returns the lowest-priority-value element first. Perfect for "process
  the most urgent ticket next."

### Sorted collections

- `SortedDictionary<TKey, TValue>`: a balanced binary tree (red-black). O(log n)
  operations; keys always in order.
- `SortedList<TKey, TValue>`: two sorted arrays. Less memory, O(log n) lookups, but
  O(n) inserts.
- `SortedSet<T>`: a sorted tree of unique values, with range queries (`GetViewBetween`).

### `LinkedList<T>`

A doubly linked list. O(1) insert/remove *when you already hold the node*, but O(n) to
find anything, poor cache behavior and an allocation per node. It's rarely the right
choice; an LRU cache (dictionary plus linked list) is the classic legitimate use.

### Choosing

| Your code mostly... | Use |
|---|---|
| Iterates and appends | `List<T>` |
| Looks up by key | `Dictionary<TKey, TValue>` |
| Checks membership / removes duplicates | `HashSet<T>` |
| Processes items in arrival order | `Queue<T>` (or `Channel<T>` across threads, Chapter 10) |
| Processes the most important item next | `PriorityQueue<TElement, TPriority>` |
| Needs items always sorted by key | `SortedDictionary` / `SortedSet` |
| Needs fixed-size, high-performance storage | `T[]` / `Span<T>` |

---

## 4. Collection interfaces: what to accept and return

.NET has a layered set of collection interfaces:

```text
IEnumerable<T>                   can be iterated (once, lazily, maybe)
 └─ IReadOnlyCollection<T>       + Count
     ├─ IReadOnlyList<T>         + index access
     └─ IReadOnlySet<T>          + set queries
ICollection<T>                   + Add/Remove/Clear (mutable)
 ├─ IList<T>                     + index access, Insert/RemoveAt
 └─ ISet<T>
IDictionary<TKey,TValue> / IReadOnlyDictionary<TKey,TValue>
```

Guidelines:

- **Parameters: accept the least you need.** If a method only iterates, take
  `IEnumerable<T>`. If it needs `Count` or indexing, take `IReadOnlyList<T>` or
  `IReadOnlyCollection<T>`.
- **Return types: return something specific and honest.** Returning `IReadOnlyList<T>`
  tells callers the data is materialized, has a count, and isn't theirs to modify.
- **Be careful returning `IEnumerable<T>`.** It might be a lazy query (Chapter 7) that
  re-executes, hits a database, or throws later. Callers then defensively call
  `.ToList()`, or worse, enumerate it several times.

> **⚠️ What can go wrong:** "Multiple enumeration." A method receives an
> `IEnumerable<T>`, calls `.Count()` and then `foreach`. If the source is a LINQ query
> over a database, that's two queries. If it's a generator that reads a file, the file is
> read twice. Materialize once (`ToList()`) when you need more than one pass.

### Collection expressions

C# 12 added a uniform syntax for creating collections:

```csharp
int[] numbers = [1, 2, 3];
List<string> names = ["maria", "omar"];
ReadOnlySpan<char> vowels = ['a', 'e', 'i', 'o', 'u'];
IReadOnlyList<int> combined = [..numbers, 4, 5];   // spread
```

The compiler picks an efficient construction strategy for each target type.

---

## 5. Immutable and frozen collections

### Read-only isn't immutable

`IReadOnlyList<T>` and `list.AsReadOnly()` prevent *you* from changing the collection
through that reference. They don't stop the *owner* from changing it underneath you:

```csharp
var list = new List<int> { 1, 2 };
IReadOnlyList<int> view = list;
list.Add(3);
Console.WriteLine(view.Count);   // 3
```

### `System.Collections.Immutable`

`ImmutableList<T>`, `ImmutableDictionary<TKey, TValue>`, `ImmutableArray<T>` and friends
can **never** change. "Modifying" operations return a new collection, sharing most of
the structure with the old one:

```csharp
var v1 = ImmutableList.Create(1, 2, 3);
var v2 = v1.Add(4);      // v1 still has 3 items
```

They're ideal for state shared across threads and for snapshots. The cost: most are
tree-based, so lookups are O(log n) and slower than their mutable equivalents.
`ImmutableArray<T>` is the exception: an immutable wrapper over an array, with fast reads
and expensive "modifications" (full copy).

### Frozen collections (.NET 8+)

`FrozenDictionary<TKey, TValue>` and `FrozenSet<T>` are built once and then read many
times. Construction is slower because they analyze the keys to pick an optimal lookup
strategy, but **reads are faster than `Dictionary`**.

```csharp
private static readonly FrozenDictionary<string, TicketPriority> PriorityByLabel =
    new Dictionary<string, TicketPriority>(StringComparer.OrdinalIgnoreCase)
    {
        ["low"] = TicketPriority.Low,
        ["normal"] = TicketPriority.Normal,
        ["high"] = TicketPriority.High,
        ["urgent"] = TicketPriority.Urgent,
    }.ToFrozenDictionary(StringComparer.OrdinalIgnoreCase);
```

Perfect for configuration, lookup tables and anything loaded at startup and read on
every request.

| | Mutable | Immutable | Frozen |
|---|---|---|---|
| Can change after creation | Yes | No (returns new) | No |
| Read speed | Fast | Slower (trees) | Fastest |
| Build speed | Fast | Medium | Slow |
| Best for | Working data | Shared, evolving snapshots | Static lookup tables |

---

## 6. `Span<T>` and `Memory<T>`

### The problem

Parsing and text processing used to allocate constantly. Taking a substring allocated a
new string; slicing an array meant copying it:

```csharp
string line = "T-1042,High,Cannot log in";
string idPart = line.Substring(2, 4);   // allocates "1042"
int id = int.Parse(idPart);
```

At thousands of requests per second, these small allocations add up to real GC pressure
(Chapter 11).

### The solution: a view over memory

`Span<T>` is a **view** over a contiguous region of memory: an array, part of an array,
a string's characters, stack memory, or native memory. Slicing a span creates another
view; nothing is copied.

```csharp
ReadOnlySpan<char> line = "T-1042,High,Cannot log in";
ReadOnlySpan<char> idPart = line.Slice(2, 4);   // no allocation: just a view
int id = int.Parse(idPart);                     // parses directly from the span
```

Internally a span is just a reference to the start plus a length. Most of the BCL
(`int.Parse`, `Utf8Formatter`, `Stream.Read`, `string.Create`) has span-based overloads.

### The restriction: `ref struct`

`Span<T>` is a **`ref struct`**: it can only live on the stack. You can't store it in a
field of a class, box it, capture it in a lambda, or (until recent C# versions relaxed
some rules) use it across an `await`. The restriction exists because a span can point at
stack memory; if it escaped to the heap, it could outlive the memory it points to.

### `Memory<T>`

When you need a span-like view that *can* live on the heap (in a field, across an
`await`), use `Memory<T>` / `ReadOnlyMemory<T>`. Call `.Span` when you're ready to do the
actual work synchronously.

```csharp
async Task ProcessAsync(ReadOnlyMemory<byte> data, CancellationToken ct)
{
    await Task.Delay(10, ct);
    Checksum(data.Span);   // use the span in synchronous code
}
```

> **🧭 When not to use it:** Spans are for hot paths: parsers, serializers, protocol
> handlers, high-throughput middleware. In ordinary business logic, `string.Substring`
> is clearer and the allocation doesn't matter. Use spans when a profiler shows
> allocation pressure, or when you're writing library code that will be called millions
> of times.

---

## 7. In practice: a ticket queue for Beacon

Beacon's support agents need a "next ticket" button that always serves the most urgent,
oldest ticket first. And the CLI should import tickets from a CSV file. Two structures
fit perfectly: `PriorityQueue` and span-based parsing.

### The triage queue

```csharp
// src/Beacon.Core/Tickets/TriageQueue.cs
namespace Beacon.Core.Tickets;

public sealed class TriageQueue
{
    private readonly PriorityQueue<Ticket, (int Urgency, DateTimeOffset CreatedAt)> _queue = new(
        Comparer<(int Urgency, DateTimeOffset CreatedAt)>.Create((a, b) =>
        {
            int byUrgency = a.Urgency.CompareTo(b.Urgency);
            return byUrgency != 0 ? byUrgency : a.CreatedAt.CompareTo(b.CreatedAt);
        }));

    private readonly HashSet<TicketId> _queued = [];

    public int Count => _queue.Count;

    public bool Enqueue(Ticket ticket)
    {
        if (!ticket.IsActive || !_queued.Add(ticket.Id))
            return false;                                  // inactive or already queued

        // PriorityQueue dequeues the LOWEST priority first, so invert urgency.
        int urgency = -(int)ticket.Priority;
        _queue.Enqueue(ticket, (urgency, ticket.CreatedAt));
        return true;
    }

    public Ticket? Next()
    {
        while (_queue.TryDequeue(out var ticket, out _))
        {
            _queued.Remove(ticket.Id);
            if (ticket.IsActive) return ticket;            // skip tickets resolved while queued
        }
        return null;
    }
}
```

Points worth noticing:

- **Composite priority** as a tuple: urgency first, then age. `ValueTuple` is a struct,
  so no allocations per entry.
- **`HashSet<TicketId>` for O(1) duplicate checks.** `PriorityQueue` has no efficient
  `Contains`; pairing structures is a common technique.
- **Lazy deletion.** Removing an arbitrary element from a heap is expensive, so instead
  we skip stale entries when dequeuing.

### Span-based CSV import

```csharp
// src/Beacon.Cli/TicketCsvParser.cs
using Beacon.Core.Tickets;

internal static class TicketCsvParser
{
    // Format: id,priority,title    e.g.  1042,High,Cannot log in
    public static bool TryParse(ReadOnlySpan<char> line, DateTimeOffset now, out Ticket? ticket)
    {
        ticket = null;

        int first = line.IndexOf(',');
        if (first < 0) return false;
        int second = line[(first + 1)..].IndexOf(',');
        if (second < 0) return false;
        second += first + 1;

        if (!int.TryParse(line[..first], out int id)) return false;
        if (!Enum.TryParse(line[(first + 1)..second], ignoreCase: true, out TicketPriority priority))
            return false;

        var title = line[(second + 1)..].Trim();
        if (title.IsEmpty) return false;

        ticket = new Ticket(new TicketId(id), title.ToString(), priority, now);
        return true;
    }
}
```

Only one string is allocated per line, the title, because the `Ticket` needs to keep it.
The ID and priority are parsed directly from slices. (The title rule in the `Ticket`
constructor still applies; validation lives in one place.) Range syntax like
`line[..first]` is shorthand for `Slice`.

Wire it up in `Program.cs`:

```csharp
var queue = new TriageQueue();
foreach (var line in File.ReadLines("tickets.csv"))
{
    if (TicketCsvParser.TryParse(line, DateTimeOffset.UtcNow, out var ticket))
        queue.Enqueue(ticket!);
}

while (queue.Next() is { } next)
    Console.WriteLine($"{next.Priority,-7} {next.Id} {next.Title}");
```

Note `File.ReadLines` (lazy, one line at a time) rather than `File.ReadAllLines` (loads
the whole file into an array). For a large import, that's the difference between
constant memory and memory proportional to file size.

---

## 8. What can go wrong

- **O(n) inside a loop** → O(n²). `List.Contains`, `Remove`, `IndexOf`, LINQ `First`,
  `Any` inside `foreach`. *Fix:* build a `HashSet` or `Dictionary` once, outside the loop.
- **Mutating a collection while iterating it** throws `InvalidOperationException`
  ("Collection was modified"). *Fix:* iterate a copy, collect changes and apply after,
  or iterate backwards by index when removing from a list.
- **Bad hash codes** degrade dictionaries to linear scans. Mutable keys break them
  entirely.
- **Culture-sensitive string keys.** Use `StringComparer.Ordinal` or
  `OrdinalIgnoreCase` for keys.
- **Unbounded growth.** An in-memory cache or queue with no size limit is a memory leak
  with extra steps.
- **Sharing mutable collections across threads.** `List<T>` and `Dictionary<TKey,
  TValue>` aren't thread-safe for concurrent writes; corruption can be silent. See
  Chapter 10 for concurrent collections.
- **`ToList()` everywhere "just in case."** Each call copies; in a hot path, that's
  needless allocation.

---

## 9. How an experienced engineer thinks about this

- **Ask "what does my code do most?"** Lookups, appends, ordered traversal, membership
  checks. Pick the structure that makes *that* cheap.
- **Watch for hidden loops.** A method call can be a loop. Know which ones are.
- **Prefer the simple default.** `List<T>` and `Dictionary<TKey, TValue>` cover the vast
  majority of cases. Exotic structures need a reason.
- **Make intent visible in types.** A `HashSet` says "unique," a `FrozenDictionary` says
  "static lookup," an `IReadOnlyList` return type says "don't modify this."
- **Optimize allocations only where it matters.** Spans and pooling for hot paths;
  clarity everywhere else.

---

## 10. Check yourself

**Questions**

1. Why is `HashSet<T>.Contains` O(1) on average but `List<T>.Contains` O(n)?
2. What does "amortized O(1)" mean for `List<T>.Add`?
3. Why is `List<T>` usually faster than `LinkedList<T>` even for workloads with some
   insertions?
4. What's the difference between read-only, immutable and frozen collections?
5. Why can't `Span<T>` be stored in a class field? What do you use instead?
6. What is "multiple enumeration," and when is it dangerous?

**Exercises**

1. Benchmark (roughly, with `Stopwatch` for now) `List.Contains` vs `HashSet.Contains`
   for 10, 1,000 and 100,000 items. At what size does the `HashSet` start winning?
2. Add a method to `TriageQueue` that returns the next N tickets without removing them.
   (Hint: `UnorderedItems` plus sorting, or a copy of the queue.)
3. Rewrite `TicketCsvParser` to also support quoted titles containing commas. How does
   the span approach change?
4. Replace a hard-coded `Dictionary` lookup table in your own code with a
   `FrozenDictionary`.

**Interview-style questions**

- "How does a hash table work? What happens on a collision?"
- "When would you use a `SortedDictionary` instead of a `Dictionary`?"
- "What is `Span<T>`, and why was it introduced?"
- "Find the performance problem in this code" (an O(n) lookup in a loop, always).

---

## 11. Going deeper

- [Microsoft docs: Collections and data structures](https://learn.microsoft.com/dotnet/standard/collections/)
- [Microsoft docs: Memory and spans](https://learn.microsoft.com/dotnet/standard/memory-and-spans/)
- *Grokking Algorithms* by Aditya Bhargava — a gentle, visual introduction to data
  structures and Big-O.

**Next:** [Chapter 6 — Delegates, Events and Lambdas](06-delegates-events-and-lambdas.md)
covers how C# treats functions as values, the foundation of LINQ, events and much of
modern .NET.
