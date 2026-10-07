# Types, Values and References

Every bug report that ends with *"but I changed it — why didn't it change?"* or
*"I only changed this one — why did that one change too?"* is a misunderstanding of
values versus references. So are many `NullReferenceException`s, surprise allocations and
"why is this struct so slow" questions.

This chapter builds an accurate picture of how C# values live in memory, what copying
really means, how `null` is handled in modern C#, and how to choose between classes,
structs and records. It's the vocabulary for the rest of Book I.

---

## 1. The problem: what does `a = b` actually do?

Look at two almost identical snippets:

```csharp
var p1 = new PointStruct(1, 2);
var p2 = p1;
p2.X = 100;
Console.WriteLine(p1.X);   // 1

var c1 = new PointClass(1, 2);
var c2 = c1;
c2.X = 100;
Console.WriteLine(c1.X);   // 100
```

The only difference is that `PointStruct` is declared with `struct` and `PointClass`
with `class`. The assignment `x = y` means something different for each:

- For a **value type**, the variable *contains the data*. Assignment copies the data.
- For a **reference type**, the variable contains a *reference* (essentially an address)
  to an object somewhere else. Assignment copies the reference; both variables now point
  at the same object.

That is the whole idea. Everything else in this chapter is consequences.

---

## 2. The mental model

### The two families

| Value types | Reference types |
|---|---|
| `int`, `long`, `double`, `decimal`, `bool`, `char` | `string` |
| `DateTime`, `TimeSpan`, `Guid` | arrays (`int[]`, `Ticket[]`) |
| `enum`s | `class`es, `record` (class) |
| `struct`s, `record struct`s | delegates, `object` |
| `Nullable<T>` (`int?`) | interfaces (when holding a value) |
| tuples `(int, string)` (`ValueTuple`) | |

All types ultimately derive from `System.Object`. Value types derive from
`System.ValueType`, which the runtime treats specially.

### Where things live (and why "stack vs heap" is only half true)

You'll often hear: *"value types live on the stack, reference types live on the heap."*
It's a useful first approximation and a wrong rule. The accurate version:

- **Reference-type objects** live on the **managed heap**, where the garbage collector
  tracks them. (The JIT can occasionally avoid this through *escape analysis* in recent
  .NET versions, but you should not count on it.)
- **Value-type values live wherever their container lives.**
  - A local `int` in a method lives in that method's stack frame (or just in a CPU
    register).
  - An `int` field inside a class object lives *inside that object, on the heap*.
  - An array of `int` is a heap object containing all the ints inline.
  - A struct field inside another struct lives inline in the outer struct.

```text
  Stack frame of Main                     Managed heap
  ┌────────────────────────┐              ┌─────────────────────────────────┐
  │ int count = 3          │              │ Ticket object                   │
  │ PointStruct p = (1,2)  │              │ ┌──────────┬──────────────────┐ │
  │ Ticket t ──────────────┼────────────► │ │ header   │ method table ptr │ │
  │                        │              │ ├──────────┴──────────────────┤ │
  └────────────────────────┘              │ │ int Id = 1   (inline value) │ │
                                          │ │ string Title ──────────┐    │ │
                                          │ │ Status = Open (inline) │    │ │
                                          │ └────────────────────────┼────┘ │
                                          │ string "Cannot log in" ◄─┘      │
                                          └─────────────────────────────────┘
```

The real distinction is not *where* data lives but **whether it has identity**:

- A reference-type object has an identity: two references can point to the same object,
  and a change through one is visible through the other.
- A value has no identity. It's just bits. Copies are independent.

> **🧱 Durable:** "Does this thing have identity, or is it just a value?" is the question
> to ask when designing a type. A `Ticket` has identity (ticket #42 is ticket #42 even if
> its title changes). A `Money` amount or a `DateRange` doesn't; two `$10.00` are
> interchangeable.

### Every object carries overhead

On 64-bit .NET, each heap object has a header and a pointer to its type's *method table*:
16 bytes before any of your fields, and a minimum object size of 24 bytes. An array of a
million `PointClass` objects is a million separate allocations plus a million 8-byte
references. An array of a million `PointStruct` values is one allocation with the data
packed contiguously. This is why value types matter for performance (Chapter 11 measures
it).

---

## 3. Copy semantics in practice

### Passing to methods

By default C# passes **by value**: the method gets a copy of the variable.

```csharp
void Rename(Ticket t)    { t = t with { Title = "Renamed" }; }   // reassigns the local copy
void Bump(PointClass p)  { p.X++; }                               // mutates the shared object
void Bump(PointStruct p) { p.X++; }                               // mutates a copy
```

- Passing a reference type copies the **reference**. The method can mutate the shared
  object, but reassigning the parameter has no effect on the caller.
- Passing a value type copies the **whole value**. The method can't affect the caller's
  value at all.

### `ref`, `out`, `in` and `ref readonly`

These modifiers pass the *variable itself* (a managed pointer to it) instead of a copy:

| Modifier | Meaning | Typical use |
|---|---|---|
| `ref` | Caller's variable, read and write | Swapping, updating a large struct in place |
| `out` | Caller's variable, must be assigned by the method | `int.TryParse(s, out var n)` |
| `in` / `ref readonly` | Caller's variable, read-only | Passing large structs without copying |

```csharp
static void Swap<T>(ref T a, ref T b) => (a, b) = (b, a);

int x = 1, y = 2;
Swap(ref x, ref y);   // x == 2, y == 1
```

You'll rarely need `ref` in business code. You'll meet it constantly in
high-performance code (`Span<T>`, parsers, serializers).

### The mutable struct trap

```csharp
var tickets = new List<PointStruct> { new(1, 2) };
tickets[0].X = 5;    // compile error CS1612
```

`tickets[0]` calls the list's indexer, which *returns a copy*. Mutating a temporary copy
would silently do nothing, so the compiler refuses. With an array it works, because array
elements are variables:

```csharp
var arr = new[] { new PointStruct(1, 2) };
arr[0].X = 5;        // fine: modifies the element in place
```

The same trap appears with `readonly` fields and properties returning structs: the
compiler may make *defensive copies* to protect a readonly value, so your mutation
silently lands on the copy.

> **⚠️ What can go wrong:** Mutable structs behave differently depending on whether
> you're holding a variable, an array element, a property or a collection element. The
> standard advice: **make structs immutable** (`readonly struct`). Then copies can't
> diverge, and the compiler never needs defensive copies.

---

## 4. Boxing

A value type can be treated as an `object` or as an interface it implements. Because
`object` is a reference type, the runtime must create a heap object to hold the value.
That's **boxing**:

```csharp
int n = 42;
object o = n;            // box: allocate a heap object, copy 42 into it
int m = (int)o;          // unbox: check the type, copy the value back out
IComparable c = n;       // also boxes
```

Boxing costs an allocation and a copy, and the boxed copy is independent of the
original. It used to be everywhere: `ArrayList`, `Hashtable` and `string.Format` boxed
every value type passed to them. Generics (Chapter 4) eliminated most of it, but it
still hides in a few places:

- Calling an interface method on a struct through an interface-typed variable.
- Passing a struct to a parameter of type `object` (for example, older logging APIs that
  take `params object[] args`).
- `Enum.HasFlag` on old runtimes, `struct.Equals(object)` when the struct doesn't
  override it, and `GetHashCode` via the default `ValueType` implementation (which also
  uses reflection and is slow).

> **🧭 When not to care:** In ordinary business code a few boxes don't matter. Worry about
> boxing in hot paths, such as per-request middleware, tight loops or serializers, and
> confirm with a profiler or allocation counter before optimizing.

---

## 5. Strings: a reference type that behaves like a value

`string` is a reference type, but it's **immutable**, and `==` compares contents. So in
practice strings feel like values:

```csharp
string a = "hello";
string b = a;
b += " world";           // creates a NEW string; a is unchanged
Console.WriteLine(a);    // hello
```

Every "modification" creates a new string. That's why building a string in a loop with
`+=` is quadratic, and why `StringBuilder` and interpolated-string handlers exist:

```csharp
var sb = new StringBuilder();
foreach (var t in tickets) sb.Append(t.Id).Append(',');
```

Two more facts worth knowing:

- **Literals are interned**: identical string literals in an assembly refer to the same
  object. Strings created at run time aren't interned unless you ask.
- **Comparison needs a culture decision.** `string.Equals(a, b)` is ordinal (byte-wise
  UTF-16). Culture-sensitive comparisons (`StringComparison.CurrentCulture`) give
  different answers on different machines. For identifiers, keys, file paths and
  protocol values, use `StringComparison.Ordinal` or `OrdinalIgnoreCase`.

> **⚠️ What can go wrong:** The "Turkish I" problem. `"FILE".ToLower()` on a Turkish-culture
> machine is `"fıle"` with a dotless ı, so `== "file"` fails. Use `ToLowerInvariant()`
> or ordinal comparisons for anything that isn't text shown to humans.

---

## 6. Equality

C# has several notions of "equal", and getting them mixed up causes subtle bugs in
dictionaries, `HashSet`s, LINQ `Distinct()` and EF Core change tracking.

| Kind | Meaning | Default for |
|---|---|---|
| **Reference equality** | Same object | classes (`==`, `Equals`) |
| **Value equality** | Same contents | structs (`Equals`), records, `string` |

```csharp
var a = new PointClass(1, 2);
var b = new PointClass(1, 2);
a == b;                        // false: different objects
ReferenceEquals(a, b);         // false

var r1 = new Ticket(1, "Login", TicketStatus.Open);
var r2 = new Ticket(1, "Login", TicketStatus.Open);
r1 == r2;                      // true: records compare by value
```

Rules that keep you out of trouble:

1. If you override `Equals`, also override `GetHashCode`, and make sure **equal objects
   have equal hash codes**. Otherwise dictionaries and hash sets break silently.
2. **Don't use mutable fields in `GetHashCode`.** Mutate an object after putting it in a
   `HashSet`, and the set can no longer find it.
3. Prefer **records** when you want value equality; the compiler generates correct
   `Equals`, `GetHashCode`, `==` and `!=` for you.
4. Implement `IEquatable<T>` on structs to avoid the boxing `Equals(object)`. Records and
   record structs do this automatically.

---

## 7. `null` and nullable reference types

### Two different kinds of null

- **Nullable value types**: `int?` is shorthand for `Nullable<int>`, a struct with a
  `HasValue` flag and a `Value`. A plain `int` can never be null.
- **Reference types** have always been able to hold `null`. Dereferencing it throws
  `NullReferenceException`, historically the most common exception in .NET.

### Nullable reference types (NRT)

Since C# 8, and enabled by default in new projects since .NET 6, the compiler tracks
nullability of references *statically*:

```xml
<Nullable>enable</Nullable>
```

```csharp
string  title    = "Login";    // never null (the compiler warns if you assign null)
string? assignee = null;       // may be null

int len1 = title.Length;       // fine
int len2 = assignee.Length;    // warning CS8602: possible null dereference

if (assignee is not null)
    len2 = assignee.Length;    // fine: the compiler knows it's not null here
```

Crucially, **NRT is a compile-time feature only**. `string` and `string?` are the same
runtime type. The compiler uses annotations and flow analysis to warn you; at run time,
a `string` can still be null if it came from code that ignored the warnings, from
deserialization, or from reflection.

Tools you'll use:

- `?.` and `??`: `assignee?.Length ?? 0`.
- `ArgumentNullException.ThrowIfNull(arg)` at public API boundaries.
- `required` members: `public required string Title { get; init; }` forces callers to
  set it in the object initializer.
- Attributes such as `[NotNullWhen(true)]` to teach the compiler about your
  `TryGet`-style methods.
- `!` (the "null-forgiving" operator) tells the compiler "trust me." Every `!` is a claim
  you can't prove to the compiler; use it rarely and deliberately.

> **🧱 Durable:** Treat nullability warnings as errors
> (`<WarningsAsErrors>nullable</WarningsAsErrors>`) in new code. The point of NRT is that
> "can this be null?" becomes part of every signature, which is a design decision, not a
> warning to suppress.

---

## 8. Classes, structs and records: choosing

C# gives you five ways to declare a data-carrying type. Here's how to choose.

| Declaration | Kind | Equality | Typical use |
|---|---|---|---|
| `class` | reference | reference | Entities and services; anything with identity or behavior |
| `record` / `record class` | reference | value | Immutable data: DTOs, messages, events, value objects |
| `struct` | value | value (slow default) | Small, short-lived, high-volume data |
| `readonly struct` | value | value (slow default) | Same, immutable (the recommended struct form) |
| `record struct` / `readonly record struct` | value | value (fast, generated) | Small value objects: `Money`, `Coordinates`, IDs |

### Guidelines for structs

Microsoft's long-standing guidance, which still holds: consider a struct when the type

- logically represents a **single value** (like a number or a date),
- is **small** (rule of thumb: 16 bytes or less, because every assignment copies it),
- is **immutable**, and
- won't be **boxed** frequently.

Larger structs aren't wrong, but you'll want to pass them with `in` and measure.

### Records

```csharp
public sealed record Ticket(int Id, string Title, TicketStatus Status);

var t2 = t1 with { Status = TicketStatus.Resolved };   // non-destructive mutation
```

A positional record gives you: init-only properties, a constructor, value equality,
a readable `ToString()`, deconstruction, and `with` expressions. It's ideal for data
that moves between layers: API requests and responses, messages, events.

> **🧭 When not to use records:** For EF Core entities, use classes. Entities have
> identity (two rows with the same values are still two rows), EF Core tracks them by
> reference, and value equality confuses the change tracker. Records are for values and
> messages, not for things that change over time.

---

## 9. In practice: Beacon's types

Chapter 1 created `Ticket` as a record. That was fine for a list of sample data, but a
ticket is an *entity*: it has identity and its status changes over time. Let's model
the domain properly, using each kind of type where it fits.

### A strongly typed ID

Using `int` for every ID lets you pass a `UserId` where a `TicketId` is expected, and the
compiler won't notice. A tiny `readonly record struct` fixes that for free: no heap
allocation, generated equality, and a distinct type.

```csharp
// src/Beacon.Core/Tickets/TicketId.cs
namespace Beacon.Core.Tickets;

public readonly record struct TicketId(int Value)
{
    public override string ToString() => $"T-{Value}";
}
```

### The entity

```csharp
// src/Beacon.Core/Tickets/Ticket.cs
namespace Beacon.Core.Tickets;

public enum TicketStatus { Open, InProgress, Resolved, Closed }

public sealed class Ticket
{
    public Ticket(TicketId id, string title, DateTimeOffset createdAt)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(title);
        Id = id;
        Title = title.Trim();
        CreatedAt = createdAt;
        Status = TicketStatus.Open;
    }

    public TicketId Id { get; }
    public string Title { get; private set; }
    public TicketStatus Status { get; private set; }
    public string? Assignee { get; private set; }
    public DateTimeOffset CreatedAt { get; }

    public bool IsActive => Status is TicketStatus.Open or TicketStatus.InProgress;

    public void AssignTo(string assignee)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(assignee);
        Assignee = assignee;
        if (Status == TicketStatus.Open) Status = TicketStatus.InProgress;
    }

    public void Resolve()
    {
        if (Status == TicketStatus.Closed)
            throw new InvalidOperationException($"Ticket {Id} is closed.");
        Status = TicketStatus.Resolved;
    }
}
```

Notice the decisions:

- **`class`, not `record`**: a ticket has identity and changes over time.
- **Private setters**: state changes only through methods that enforce the rules
  (`Resolve` refuses closed tickets). Chapter 3 discusses this encapsulation in depth.
- **`string? Assignee`**: the signature itself says "a ticket may be unassigned."
  Any code that reads `Assignee` must deal with that.
- **`DateTimeOffset` rather than `DateTime`**: it carries the UTC offset, which avoids
  a whole class of time-zone bugs. It's a value type, so storing it costs nothing extra.
- **`sealed`**: nothing should inherit from `Ticket`, and sealing lets the JIT
  devirtualize calls (Chapter 1).

### A value object for the API

When Beacon gets an API (Book III), clients will receive a summary, not the entity
itself. That's a perfect record:

```csharp
// src/Beacon.Core/Tickets/TicketSummary.cs
namespace Beacon.Core.Tickets;

public sealed record TicketSummary(TicketId Id, string Title, TicketStatus Status, string? Assignee)
{
    public static TicketSummary From(Ticket t) => new(t.Id, t.Title, t.Status, t.Assignee);
}
```

Update `Program.cs` in `Beacon.Cli` to use the new types:

```csharp
using Beacon.Core.Tickets;

var now = DateTimeOffset.UtcNow;
var tickets = new List<Ticket>
{
    new(new TicketId(1), "Cannot log in", now),
    new(new TicketId(2), "Export to PDF is slow", now),
    new(new TicketId(3), "Typo on the pricing page", now),
};

tickets[1].AssignTo("maria");
tickets[2].Resolve();

foreach (var s in tickets.Select(TicketSummary.From))
    Console.WriteLine($"{s.Id,-5} {s.Status,-10} {s.Assignee ?? "-",-6} {s.Title}");
```

```text
T-1   Open       -      Cannot log in
T-2   InProgress maria  Export to PDF is slow
T-3   Resolved   -      Typo on the pricing page
```

Note `tickets[1].AssignTo("maria")` works even though `List<T>`'s indexer returns a copy:
the copy is a *reference*, so it points at the same `Ticket` object. That is exactly the
difference from the struct example in section 3.

---

## 10. What can go wrong

- **Accidental sharing.** Two parts of the system hold a reference to the same mutable
  object (a cached list, a shared settings object), and one mutates it. *Prevention:*
  immutability, defensive copies at boundaries, read-only interfaces
  (`IReadOnlyList<T>`).
- **Lost updates on structs.** Mutating a copy (from a property, collection indexer or
  `foreach` variable) and expecting the original to change. *Prevention:* readonly
  structs.
- **`NullReferenceException` despite NRT.** Data from JSON, a database or reflection
  bypasses compile-time checks. *Prevention:* validate at boundaries (Book III covers
  validation).
- **Broken dictionaries.** Overriding `Equals` without `GetHashCode`, or mutating a key
  after inserting it.
- **Hidden allocations.** Boxing in hot paths, closures (Chapter 6) and string
  concatenation in loops.
- **Large structs copied everywhere.** A 200-byte struct passed by value through five
  method calls is 1 KB of copying. Measure; pass with `in`; or make it a class.

---

## 11. How an experienced engineer thinks about this

- **Default to classes for entities and services, records for data, and reach for
  structs deliberately.** Structs are an optimization with sharp edges; records give you
  most of the safety benefits of values without the copying concerns.
- **Make illegal states unrepresentable.** `TicketId` instead of `int`, `string?` only
  where null is genuinely allowed, private setters so only valid transitions happen.
  Every rule the type system enforces is one fewer test and one fewer production bug.
- **Immutability by default.** Immutable objects can be shared freely across threads and
  layers. Mutate only when you have a reason, and keep mutation inside the type that owns
  the rules.
- **Know the cost model, measure before optimizing.** Allocations, boxing and copying
  matter in hot paths and almost nowhere else.

---

## 12. Check yourself

**Questions**

1. What exactly is copied when you assign one reference-type variable to another?
   When you assign a struct?
2. Where does an `int` field of a class instance live in memory?
3. Why does `list[0].X = 5` fail to compile for a `List<SomeStruct>`, but work for an array?
4. What is boxing, and name two situations in modern code where it still occurs.
5. Nullable reference types are a compile-time feature. What does that mean in practice?
6. Why should EF Core entities usually be classes rather than records?
7. What are the rules for overriding `Equals` and `GetHashCode`?

**Exercises**

1. Write a `readonly record struct Money(decimal Amount, string Currency)` with an
   `Add` method that refuses to add different currencies. Write three tests in your head
   (you'll write real ones in Chapter 14).
2. Put a mutable class into a `HashSet<T>` using a field in `GetHashCode`, mutate the
   field, and try `Contains`. Explain the result.
3. Enable `<WarningsAsErrors>nullable</WarningsAsErrors>` in `Beacon.Core` and fix any
   warnings.
4. Use SharpLab to view the IL for `object o = 42;`. Find the `box` instruction.

**Interview-style questions**

- "What's the difference between a value type and a reference type?"
- "When would you use a struct instead of a class?"
- "What's a record, and when would you not use one?"
- "How do nullable reference types work, and what are their limits?"

---

## 13. Going deeper

- [Microsoft docs: Value types](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/value-types)
  and [Choosing between class and struct](https://learn.microsoft.com/dotnet/standard/design-guidelines/choosing-between-class-and-struct)
- [Microsoft docs: Nullable reference types](https://learn.microsoft.com/dotnet/csharp/nullable-references)
- [Microsoft docs: Records](https://learn.microsoft.com/dotnet/csharp/language-reference/builtin-types/record)
- *Pro .NET Memory Management* by Konrad Kokosa — the definitive book on how .NET lays
  out and manages memory.

**Next:** [Chapter 3 — Object-Oriented Design in C#](03-object-oriented-design-in-c.md)
uses these types to build behavior: encapsulation, interfaces, inheritance and
composition.
