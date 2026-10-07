# Generics

You use generics every day: `List<T>`, `Dictionary<TKey, TValue>`, `Task<T>`,
`IEnumerable<T>`. Writing good generic code yourself, and understanding what the runtime
does with it, is a different skill. It explains why `List<int>` is fast while
`ArrayList` was slow, why you can assign `IEnumerable<string>` to `IEnumerable<object>`
but not `List<string>` to `List<object>`, and how `INumber<T>` lets you write one `Sum`
method for every numeric type.

---

## 1. The problem: reuse without losing type safety

Before generics (C# 1.0), a reusable collection had to store `object`:

```csharp
var list = new ArrayList();
list.Add(42);            // boxes the int
list.Add("oops");        // compiles fine
int n = (int)list[1];    // InvalidCastException at run time
```

Two costs:

1. **No type safety.** The compiler couldn't stop you putting a string in a list of
   numbers; you found out at run time.
2. **Boxing.** Every value type was boxed on the way in and unboxed on the way out
   (Chapter 2).

The alternative, writing `IntList`, `StringList` and `TicketList`, meant duplicating code.
Generics (C# 2.0, 2005) solve both: write the code once with a **type parameter**, and
the compiler and runtime produce a type-safe, specialized version for each use.

```csharp
var list = new List<int>();
list.Add(42);            // no boxing
list.Add("oops");        // compile error
int n = list[0];         // no cast
```

---

## 2. The mental model

### Type parameters are placeholders

```csharp
public sealed class Box<T>
{
    public Box(T value) => Value = value;
    public T Value { get; }
}

var b1 = new Box<int>(42);        // Box<int>: T = int
var b2 = new Box<string>("hi");   // Box<string>: T = string
```

`Box<T>` is an **open generic type** (a template). `Box<int>` is a **closed constructed
type**: a real type with its own identity. `typeof(Box<int>) != typeof(Box<string>)`.

### How .NET compiles generics: reification

Languages implement generics in different ways, and the difference matters:

- **Java** uses *type erasure*: `List<String>` and `List<Integer>` become the same
  `List` of `Object` at run time. Generic type information is mostly gone; value types
  must be boxed.
- **C++** templates generate a full copy of the code for every type at compile time.
- **.NET** generics are **reified**: generic types exist at run time, with full type
  information, and the JIT specializes code as needed.

The .NET JIT uses a hybrid strategy:

```text
 List<int>      ─► its own machine code, int stored inline, no boxing
 List<double>   ─► its own machine code
 List<string>   ─┐
 List<Ticket>   ─┼─► ONE shared copy of machine code (all references are pointer-sized),
 List<object>   ─┘   plus a small per-type dictionary for type-specific lookups
```

- For **value types**, the JIT generates **separate code per type**. That's why
  `List<int>` stores ints inline with no boxing and runs as fast as hand-written code.
- For **reference types**, all instantiations **share one copy** of the code, because
  every reference is the same size. This keeps code size under control.

> **🧱 Durable:** Reified generics are why `typeof(T)`, `default(T)`, `new T()` and
> `is T` work in C# but not in Java. They're also why generic code over value types
> (`Span<T>`, `Dictionary<int, T>`, generic math) can be as fast as specialized code.

---

## 3. Generic methods and type inference

Methods can have their own type parameters:

```csharp
public static T Max<T>(T a, T b) where T : IComparable<T>
    => a.CompareTo(b) >= 0 ? a : b;

Max(3, 7);              // T inferred as int
Max("apple", "pear");   // T inferred as string
```

The compiler infers `T` from the arguments. Inference works from arguments only, never
from the return type, which is why some calls need explicit type arguments:

```csharp
var t = JsonSerializer.Deserialize<Ticket>(json);   // T can't be inferred from a string
```

---

## 4. Constraints

Without constraints, the compiler only lets you do with a `T` what you can do with any
`object`. Constraints tell the compiler more about `T`, unlocking operations:

| Constraint | Meaning | Unlocks |
|---|---|---|
| `where T : class` | Reference type | `null` comparisons and assignment |
| `where T : class?` | Reference type, nullable allowed | |
| `where T : struct` | Non-nullable value type | `T?` means `Nullable<T>` |
| `where T : notnull` | Non-nullable type | Safe dictionary keys |
| `where T : unmanaged` | Value type with no references | Pointers, `stackalloc`, raw memory |
| `where T : new()` | Has public parameterless constructor | `new T()` |
| `where T : SomeBase` | Derives from a class | Base-class members |
| `where T : ISomething` | Implements an interface | Interface members, no boxing |
| `where T : allows ref struct` | Can be a `ref struct` like `Span<T>` (C# 13) | Generic code over spans |

Constraints compose:

```csharp
public interface IEntity<TId> where TId : notnull
{
    TId Id { get; }
}

public sealed class InMemoryRepository<TEntity, TId>
    where TEntity : class, IEntity<TId>
    where TId : notnull
{
    private readonly Dictionary<TId, TEntity> _items = new();

    public void Add(TEntity entity) => _items.Add(entity.Id, entity);
    public TEntity? Find(TId id) => _items.GetValueOrDefault(id);
}
```

### Interface constraints avoid boxing

A subtle but important point: calling an interface method on a value-type `T` through a
constraint does **not** box:

```csharp
static bool AreEqual<T>(T a, T b) where T : IEquatable<T> => a.Equals(b);   // no boxing
static bool AreEqualBoxed(IEquatable<int> a, int b) => a.Equals(b);       // a was boxed
```

Because the JIT specializes code for each value type, `a.Equals(b)` becomes a direct (and
often inlined) call to `int.Equals(int)`. This is the mechanism that makes generic
collections fast.

---

## 5. Variance: `in` and `out`

This compiles:

```csharp
IEnumerable<string> names = new List<string> { "a", "b" };
IEnumerable<object> objects = names;      // OK
```

This doesn't:

```csharp
List<string> names = new() { "a", "b" };
List<object> objects = names;             // error
```

Why? If the second were allowed, you could do `objects.Add(42)` and put an `int` into a
list of strings. `IEnumerable<T>` only ever *gives out* `T`s, so treating a sequence of
strings as a sequence of objects is safe. `List<T>` both takes `T`s in and gives them
out, so it can't be safe in either direction.

C# expresses this with **variance annotations** on interfaces and delegates:

- **`out T` (covariance):** `T` only appears in output positions. `IEnumerable<out T>`,
  `IReadOnlyList<out T>`, `Func<out TResult>`. A `Producer<Derived>` can be used as a
  `Producer<Base>`.
- **`in T` (contravariance):** `T` only appears in input positions. `IComparer<in T>`,
  `Action<in T>`. A `Consumer<Base>` can be used as a `Consumer<Derived>`. If something
  can compare any two `object`s, it can certainly compare two `string`s.

```csharp
public interface INotificationHandler<in TNotification>
{
    Task HandleAsync(TNotification notification, CancellationToken ct);
}
```

Variance only applies to **interfaces and delegates**, and only to **reference types**
(`IEnumerable<int>` isn't an `IEnumerable<object>`, because that would require boxing
each element).

> **⚠️ What can go wrong:** Arrays are covariant for historical reasons:
> `object[] arr = new string[1]; arr[0] = 42;` compiles and throws
> `ArrayTypeMismatchException` at run time. The runtime has to check every write to a
> reference-type array, which is one reason `List<T>` and spans are preferred in modern code.

---

## 6. Generic math (static abstract members)

For years you couldn't write one `Sum` for `int`, `long`, `double` and `decimal`, because
there was no way to say "`T` supports `+`". C# 11 / .NET 7 added **static abstract
members in interfaces**, and the BCL used them to define numeric interfaces such as
`INumber<T>`, `IAdditionOperators<TSelf, TOther, TResult>` and so on.

```csharp
using System.Numerics;

public static T Sum<T>(IEnumerable<T> values) where T : INumber<T>
{
    T total = T.Zero;
    foreach (var v in values) total += v;
    return total;
}

Sum([1, 2, 3]);            // 6 (int)
Sum([1.5, 2.5]);           // 4.0 (double)
Sum([10.00m, 0.99m]);      // 10.99 (decimal)
```

Because each numeric type is a value type, the JIT generates specialized code for each,
just as fast as hand-written versions.

You'll write generic math rarely in business code. It's essential knowledge for
libraries, and for understanding modern BCL signatures.

---

## 7. Generic design guidelines

### Name type parameters meaningfully

`T` is fine for a single obvious parameter. With more than one, use descriptive names
with a `T` prefix: `TKey`, `TValue`, `TEntity`, `TResult`.

### Don't make things generic speculatively

> **🧭 When not to use it:** Generics add cognitive load. A `GenericRepository<TEntity,
> TKey, TContext, TFilter>` that handles every entity "uniformly" usually ends up with
> flags and special cases for the entities that aren't uniform. Write the concrete code
> first; extract a generic version when you have two or three real uses that are truly
> the same shape.

### Avoid the "generic repository over EF Core" trap

A well-known anti-pattern deserves a specific mention. EF Core's `DbSet<T>` *already is*
a generic repository. Wrapping it in `IRepository<T>` with `GetAll()`, `Find()`,
`Add()` usually:

- hides EF Core's real capabilities (projections, `Include`, `AsNoTracking`),
- encourages `GetAll().Where(...)` patterns that load whole tables into memory, and
- adds a layer without adding meaning.

Specific repositories (`ITicketRepository` with `FindOpenByAssigneeAsync`) or using
`DbContext` directly are usually better. Book IV, Chapter 7 discusses this in depth.

### Consider `static` caches per type

A useful trick: static fields in a generic class are **per closed type**. That gives you a
cheap, thread-safe cache keyed by type:

```csharp
internal static class TypeName<T>
{
    public static readonly string Value = typeof(T).Name;   // computed once per T
}
```

---

## 8. In practice: a generic `Result<T>` for Beacon

Beacon needs a consistent way to say "this operation succeeded with a value, or failed
with a reason," without throwing exceptions for expected failures (Chapter 8 discusses
when to use which). A small generic type does it:

```csharp
// src/Beacon.Core/Common/Result.cs
namespace Beacon.Core.Common;

public sealed record Error(string Code, string Message)
{
    public static Error NotFound(string what) => new("not_found", $"{what} was not found.");
    public static Error Conflict(string message) => new("conflict", message);
    public static Error Validation(string message) => new("validation", message);
}

public readonly struct Result<T>
{
    private readonly T? _value;

    private Result(T value) { _value = value; Error = null; }
    private Result(Error error) { _value = default; Error = error; }

    public Error? Error { get; }
    public bool IsSuccess => Error is null;

    public T Value => IsSuccess
        ? _value!
        : throw new InvalidOperationException($"Result failed: {Error!.Code}");

    public static Result<T> Success(T value) => new(value);
    public static Result<T> Failure(Error error) => new(error);

    public static implicit operator Result<T>(T value) => Success(value);
    public static implicit operator Result<T>(Error error) => Failure(error);

    public TOut Match<TOut>(Func<T, TOut> onSuccess, Func<Error, TOut> onFailure)
        => IsSuccess ? onSuccess(_value!) : onFailure(Error!);
}
```

Design decisions:

- **`readonly struct`**: returned from many methods, small (one reference plus one `T`),
  immutable. No allocation for the result itself.
- **Implicit conversions** let methods `return ticket;` or `return Error.NotFound(...)`
  naturally.
- **`Match`** forces callers to handle both cases. It's a generic method (`TOut`) on a
  generic type (`T`).

And a generic in-memory store Beacon can use until the database arrives in Book IV:

```csharp
// src/Beacon.Core/Common/IEntity.cs
namespace Beacon.Core.Common;

public interface IEntity<out TId> where TId : notnull
{
    TId Id { get; }
}
```

Make `Ticket` implement it (`public sealed class Ticket : IEntity<TicketId>`; it already
has the `Id` property). Then:

```csharp
// src/Beacon.Cli/InMemoryStore.cs
using Beacon.Core.Common;

internal sealed class InMemoryStore<TEntity, TId>
    where TEntity : class, IEntity<TId>
    where TId : notnull
{
    private readonly Dictionary<TId, TEntity> _items = new();

    public Result<TEntity> Add(TEntity entity) =>
        _items.TryAdd(entity.Id, entity)
            ? entity
            : Error.Conflict($"{typeof(TEntity).Name} {entity.Id} already exists.");

    public Result<TEntity> Get(TId id) =>
        _items.TryGetValue(id, out var entity)
            ? entity
            : Error.NotFound($"{typeof(TEntity).Name} {id}");

    public IReadOnlyCollection<TEntity> All => _items.Values;
}
```

Using it:

```csharp
var store = new InMemoryStore<Ticket, TicketId>();
store.Add(new Ticket(new TicketId(1), "Cannot log in", TicketPriority.High, DateTimeOffset.UtcNow));

var message = store.Get(new TicketId(99)).Match(
    t => $"Found {t.Title}",
    e => $"Error {e.Code}: {e.Message}");

Console.WriteLine(message);   // Error not_found: Ticket T-99 was not found.
```

`TicketId` is a `readonly record struct`, so as a dictionary key it gets generated,
non-boxing equality and hashing. Everything from Chapter 2 and this chapter is working
together.

Note that this store lives in `Beacon.Cli`, not `Beacon.Core`. It's a temporary
infrastructure detail. We're not building a "generic repository" for production; Book IV
replaces it with EF Core.

---

## 9. What can go wrong

- **Over-generalization.** Generic abstractions with many type parameters and
  constraints that are harder to understand than three concrete classes would be.
- **Code bloat with many value-type instantiations.** Each value type gets its own
  machine code. Usually fine; occasionally matters for Native AOT binary size.
- **Static state surprise.** Each closed type has its own statics. A "global" counter in
  `Cache<T>` is actually one counter per `T`.
- **Variance confusion.** Expecting `List<Derived>` to be a `List<Base>`.
- **Losing type information to `object`.** Methods that take `object` instead of a
  generic `T` reintroduce boxing and casts.
- **Reflection over generics** (Chapter 12) is notoriously awkward: `MakeGenericType`,
  open vs closed types, and AOT incompatibilities.

---

## 10. How an experienced engineer thinks about this

- **Generics are for algorithms and containers that are truly type-independent.**
  Collections, results, caches, pipelines and math. Business logic is rarely one of those.
- **Constraints are documentation.** `where TId : notnull` says something important about
  the design.
- **The rule of three applies.** Duplicate once; generalize when the third case appears
  and you can see what's actually common.
- **Know the runtime model.** Value-type specialization explains the performance of
  generic code; shared reference-type code explains why that's not a concern for
  `List<Ticket>`.

---

## 11. Check yourself

**Questions**

1. What two problems did generics solve compared to `ArrayList`?
2. How does the .NET JIT handle `List<int>` vs `List<string>` differently? Why?
3. Why is `IEnumerable<string>` assignable to `IEnumerable<object>` but `List<string>`
   isn't assignable to `List<object>`?
4. What does `where T : IEquatable<T>` buy you on a value type, besides the method?
5. What is generic math, and which C# feature made it possible?
6. Why is a generic repository over EF Core often an anti-pattern?

**Exercises**

1. Add a `Map<TOut>(Func<T, TOut>)` method to `Result<T>` that transforms a successful
   value and passes errors through unchanged.
2. Write `T Clamp<T>(T value, T min, T max) where T : INumber<T>` and test it with
   `int` and `decimal`.
3. Create `interface IHandler<in T>` and show that an `IHandler<object>` can be passed
   where an `IHandler<string>` is expected.
4. Demonstrate the per-type static field behavior with a generic class that counts
   instances.

**Interview-style questions**

- "How are .NET generics different from Java generics?"
- "Explain covariance and contravariance with an example."
- "What constraints can you put on a generic type parameter?"

---

## 12. Going deeper

- [Microsoft docs: Generics in .NET](https://learn.microsoft.com/dotnet/standard/generics/)
- [Microsoft docs: Covariance and contravariance in generics](https://learn.microsoft.com/dotnet/standard/generics/covariance-and-contravariance)
- [Microsoft docs: Generic math](https://learn.microsoft.com/dotnet/standard/generics/math)

**Next:** [Chapter 5 — Collections and Data Structures](05-collections-and-data-structures.md)
looks at the generic collections you use every day, and how to choose between them.
