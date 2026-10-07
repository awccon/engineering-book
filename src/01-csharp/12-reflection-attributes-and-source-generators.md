# Reflection, Attributes and Source Generators

How does ASP.NET Core know that `[HttpGet("{id}")]` on a method means "handle GET
requests to this URL"? How does `System.Text.Json` serialize a class it has never seen?
How does xUnit find your tests? The answer for most of .NET's history was **reflection**:
code inspecting other code's metadata at run time. Increasingly, the answer is **source
generators**: code that writes code at compile time.

You'll rarely write reflection in application code. But almost every framework you use is
built on it, and understanding both approaches explains framework behavior, startup
costs, and why Native AOT has the restrictions it does.

---

## 1. The problem: code that works with types it doesn't know

Some code needs to operate on types that didn't exist when it was written:

- A **serializer** that turns any object into JSON.
- A **DI container** that constructs any class by looking at its constructor.
- A **test runner** that finds every method marked `[Fact]`.
- An **ORM** that maps any class to a database table.
- A **web framework** that routes requests to any controller method.

Each needs to ask questions about a type: what properties do you have, what does your
constructor need, which methods carry this marker? Chapter 1 explained that every .NET
assembly carries rich **metadata** describing its types. Reflection is the API for
reading that metadata, and using it to create objects and call members dynamically.

---

## 2. The mental model: metadata at run time

```text
  Beacon.Core.dll
  ┌──────────────────────────────────────────┐
  │ Metadata tables                          │
  │  TypeDef: Ticket                         │    typeof(Ticket)
  │   ├─ Property: Title (string)            │ ─────────────────►  Type object
  │   ├─ Method: Resolve(DateTimeOffset)     │                     ├─ GetProperties()
  │   └─ CustomAttribute: [Audited]          │                     ├─ GetMethods()
  │ IL for each method                       │                     └─ GetCustomAttributes()
  └──────────────────────────────────────────┘
```

The entry points:

```csharp
Type t1 = typeof(Ticket);                 // compile-time known type
Type t2 = ticket.GetType();               // run-time type of an object
Type? t3 = Type.GetType("Beacon.Core.Tickets.Ticket, Beacon.Core");   // by name
Assembly asm = typeof(Ticket).Assembly;   // the containing assembly
```

### Inspecting

```csharp
foreach (PropertyInfo p in typeof(Ticket).GetProperties(BindingFlags.Public | BindingFlags.Instance))
    Console.WriteLine($"{p.Name}: {p.PropertyType.Name} (writable: {p.SetMethod?.IsPublic == true})");
```

```text
Id: TicketId (writable: False)
Title: String (writable: False)
Priority: TicketPriority (writable: False)
...
```

### Acting

```csharp
object? value = typeof(Ticket).GetProperty("Title")!.GetValue(ticket);
typeof(Ticket).GetMethod("Resolve")!.Invoke(ticket, [DateTimeOffset.UtcNow]);
object instance = Activator.CreateInstance(typeof(ConsoleNotifier))!;
```

Reflection can even reach private members (`BindingFlags.NonPublic`), which is how some
serializers and test tools work, and why reflection can break encapsulation.

---

## 3. Attributes: declarative metadata

An **attribute** attaches extra metadata to code. It does nothing by itself; it's data
that some other code reads via reflection (or a source generator reads at compile time).

```csharp
[AttributeUsage(AttributeTargets.Property)]
public sealed class SensitiveAttribute : Attribute;

public sealed record UserProfile(
    string Login,
    [property: Sensitive] string Email,
    [property: Sensitive] string Phone);
```

Some code can then honor it, for example a log formatter that masks sensitive properties:

```csharp
static string Describe(object obj)
{
    var parts = obj.GetType()
        .GetProperties()
        .Select(p => p.GetCustomAttribute<SensitiveAttribute>() is not null
            ? $"{p.Name}=***"
            : $"{p.Name}={p.GetValue(obj)}");
    return string.Join(", ", parts);
}

Describe(new UserProfile("maria", "maria@example.com", "+1-555-0100"));
// Login=maria, Email=***, Phone=***
```

Attributes you use constantly, and who reads them:

| Attribute | Read by |
|---|---|
| `[HttpGet]`, `[Route]`, `[FromBody]` | ASP.NET Core routing and model binding |
| `[Required]`, `[MaxLength]` | Validation, EF Core |
| `[JsonPropertyName]`, `[JsonIgnore]` | `System.Text.Json` |
| `[Fact]`, `[Theory]` | xUnit |
| `[Obsolete]` | The compiler |
| `[CallerMemberName]`, `[CallerArgumentExpression]` | The compiler |
| `[NotNullWhen]`, `[MemberNotNull]` | The compiler's nullable analysis |

Note the last three groups: some attributes are read by the **compiler**, not at run time.

---

## 4. The costs of reflection

Reflection is powerful and slow, in several ways:

1. **Run-time speed.** `PropertyInfo.GetValue` is many times slower than a direct property
   access: it validates arguments, boxes value types and goes through general-purpose
   invocation machinery. (.NET 7+ made it much faster, but the gap remains.)
2. **Startup cost.** Scanning assemblies for types and attributes at startup takes time,
   proportional to how many types there are.
3. **No compile-time safety.** `GetProperty("Titel")` compiles and returns null at run
   time. Renaming a property breaks reflection-based code silently.
4. **Trimming and AOT incompatibility.** The trimmer (Chapter 1) removes code it can't see
   being used. If a type is only ever created by `Activator.CreateInstance(typeName)`, the
   trimmer can't know, and removes it. Native AOT can't generate new code at run time, so
   `Reflection.Emit` doesn't work.

Frameworks mitigated the speed problem by **caching**: reflect once, then compile a fast
delegate (with expression trees or `Reflection.Emit`) and reuse it. That's why
serializers are slow on the first call for each type and fast afterwards. But caching
doesn't fix startup cost, safety, or AOT.

---

## 5. Source generators: doing it at compile time

A **source generator** is a component that runs *inside the compiler*. It inspects your
code (syntax trees and semantic model) and **adds new C# source files** to the
compilation. The generated code is compiled together with yours.

```text
 Your code ──► Roslyn ──► (source generators inspect it, emit more C#) ──► Roslyn compiles all ──► IL
```

The same problems reflection solved at run time are solved at build time:

| Concern | Reflection (run time) | Source generator (compile time) |
|---|---|---|
| Speed | Slow first call, cached afterwards | As fast as hand-written code |
| Startup | Scans types at startup | Nothing to scan |
| Errors | Discovered at run time | Compiler errors and warnings |
| Trimming / AOT | Problematic | Fully compatible |
| Debugging | Opaque | Generated code is visible and debuggable |

Generators you'll meet in modern .NET:

- **`System.Text.Json`** source generation (`JsonSerializerContext`): serialization code
  for your types, required for AOT.
- **`[LoggerMessage]`**: high-performance logging methods with no boxing or parsing of
  the template at run time.
- **`[GeneratedRegex]`**: regular expressions compiled to C# at build time.
- **ASP.NET Core Request Delegate Generator**: generates endpoint glue for minimal APIs
  under AOT.
- **`[LibraryImport]`**: P/Invoke marshaling code.
- Configuration binding source generator, and many third-party ones (Mapperly for object
  mapping, for example).

### Using them

```csharp
// JSON
[JsonSerializable(typeof(TicketSummary))]
[JsonSerializable(typeof(List<TicketSummary>))]
internal sealed partial class BeaconJsonContext : JsonSerializerContext;

var json = JsonSerializer.Serialize(summary, BeaconJsonContext.Default.TicketSummary);

// Regex
internal static partial class Patterns
{
    [GeneratedRegex(@"^T-(\d+)$")]
    public static partial Regex TicketReference();
}

// Logging
internal static partial class Log
{
    [LoggerMessage(Level = LogLevel.Information, Message = "Ticket {TicketId} assigned to {Assignee}")]
    public static partial void TicketAssigned(ILogger logger, TicketId ticketId, string assignee);
}
```

The pattern is always the same: you write a `partial` declaration with an attribute; the
generator writes the other half of the `partial`. In Visual Studio or Rider, expand
**Dependencies → Analyzers → (generator name)** to read the generated code.

Writing your own source generator is an advanced topic (incremental generators, Roslyn
APIs). It's worth doing when you have repetitive boilerplate across many types that
reflection would otherwise handle. For most teams, *using* existing generators is what
matters.

---

## 6. When reflection is still the right tool

> **🧭 When not to use source generators:** Reflection remains appropriate for genuinely
> dynamic scenarios: plugin systems that load assemblies at run time, admin or diagnostic
> tools that inspect arbitrary objects, test utilities, and one-off scripts. If the set
> of types isn't known at compile time, a generator can't help.

And when you write application code, ask whether you need either. Most "I need reflection"
moments in business code are better solved with interfaces, generics, or a dictionary of
delegates:

```csharp
// Reflection-based: find a handler class by naming convention. Fragile, slow, AOT-hostile.
var handlerType = Type.GetType($"Beacon.Handlers.{command.GetType().Name}Handler");

// Explicit: a registry built at startup. Fast, safe, discoverable with "Find references".
var handlers = new Dictionary<Type, Func<object, Task>> { [typeof(ResolveTicket)] = c => Resolve((ResolveTicket)c) };
```

---

## 7. In practice: Beacon's sensitive-data masking and fast JSON

Two practical additions to Beacon, one using attributes plus reflection (appropriate for a
diagnostic tool), one using source generation (appropriate for a hot path).

### A `[Sensitive]` attribute for audit details

Beacon's audit log (Chapter 11) records details of changes. Users' email addresses
shouldn't end up in it.

```csharp
// src/Beacon.Core/Common/SensitiveAttribute.cs
namespace Beacon.Core.Common;

[AttributeUsage(AttributeTargets.Property | AttributeTargets.Parameter)]
public sealed class SensitiveAttribute : Attribute;
```

```csharp
// src/Beacon.Core/Auditing/AuditFormatter.cs
using System.Collections.Concurrent;
using System.Reflection;
using Beacon.Core.Common;

namespace Beacon.Core.Auditing;

public static class AuditFormatter
{
    // Reflect once per type, then reuse: the standard caching technique.
    private static readonly ConcurrentDictionary<Type, (PropertyInfo Prop, bool Sensitive)[]> Cache = new();

    public static string Format(object value)
    {
        var props = Cache.GetOrAdd(value.GetType(), static t => t
            .GetProperties(BindingFlags.Public | BindingFlags.Instance)
            .Where(p => p.GetIndexParameters().Length == 0)
            .Select(p => (p, p.IsDefined(typeof(SensitiveAttribute), inherit: true)))
            .ToArray());

        return string.Join(", ", props.Select(x =>
            x.Sensitive ? $"{x.Prop.Name}=***" : $"{x.Prop.Name}={x.Prop.GetValue(value)}"));
    }
}
```

Note `static t => ...`: a non-capturing lambda, so `GetOrAdd` allocates nothing on cache
hits (Chapter 6). Audit formatting happens a few times per request, so reflection with a
cache is a sensible trade-off here. If the trimmer complains (for AOT), we'd switch to a
generator or an explicit interface.

### Source-generated JSON for the CLI's export

The CLI gains an `export` command that writes all tickets as JSON. We use the JSON source
generator so the CLI stays AOT-compatible (Chapter 1's `PublishAot` experiment):

```csharp
// src/Beacon.Cli/BeaconJsonContext.cs
using System.Text.Json.Serialization;
using Beacon.Core.Tickets;

[JsonSourceGenerationOptions(WriteIndented = true, PropertyNamingPolicy = JsonKnownNamingPolicy.CamelCase,
    UseStringEnumConverter = true)]
[JsonSerializable(typeof(List<TicketSummary>))]
internal sealed partial class BeaconJsonContext : JsonSerializerContext;
```

```csharp
// Program.cs (excerpt)
var summaries = tickets.Select(TicketSummary.From).ToList();
await using var file = File.Create("tickets.json");
await JsonSerializer.SerializeAsync(file, summaries, BeaconJsonContext.Default.ListTicketSummary, ct);
```

Publish the CLI with `-p:PublishAot=true` and check that there are no trim warnings. If
you'd used `JsonSerializer.Serialize(summaries)` without the context, the build would warn
that reflection-based serialization may break under AOT.

`TicketId` is a record struct with a `Value` property, so it serializes as
`{"value": 7}`. In Book III we'll add a custom converter so it appears as `"T-7"`.

---

## 8. What can go wrong

- **Silent breakage on rename.** String-based reflection (`GetProperty("Title")`) fails
  at run time after a refactoring. Use `nameof(Ticket.Title)` at minimum.
- **Slow paths.** Uncached reflection in a hot path.
- **Trim/AOT failures.** Types or members used only via reflection are removed; publish
  works, but the app fails at run time. Treat trim warnings as errors.
- **Breaking encapsulation.** Reflection that sets private fields bypasses invariants.
- **Assembly scanning at startup** making cold starts slow in large apps.
- **Hidden magic.** Convention-based reflection ("any class ending in `Handler` is
  registered") makes behavior hard to discover. A new developer can't find where something
  is wired up.

---

## 9. How an experienced engineer thinks about this

- **Prefer compile-time over run-time.** If the type set is known at build time, a source
  generator (or plain code) beats reflection on speed, safety and AOT compatibility.
- **Reflection belongs in frameworks and tools**, rarely in business logic.
- **Explicit beats magic** for application wiring. A registration list someone can read
  is worth more than a clever convention.
- **When you must reflect, cache.**

---

## 10. Check yourself

**Questions**

1. What metadata does reflection read, and where does it come from?
2. Does an attribute do anything by itself? Who acts on it?
3. Name four costs of reflection.
4. How does a source generator work, and why is it AOT-compatible?
5. Why do serializers often get faster after the first call for a type?
6. When is reflection still appropriate?

**Exercises**

1. Write a method that lists every public method in `Beacon.Core` that takes a
   `CancellationToken` but doesn't end with `Async`. (A real convention check, the kind of
   thing architecture tests do in Chapter 14.)
2. Benchmark `PropertyInfo.GetValue` vs direct property access vs a cached compiled
   delegate (`Expression.Lambda<Func<Ticket, string>>`). Use BenchmarkDotNet.
3. Convert an existing `Regex` in your code to `[GeneratedRegex]` and read the generated
   source.
4. Add a `[LoggerMessage]` method for "ticket resolved" and look at what it generates.

**Interview-style questions**

- "What is reflection, and when would you use it?"
- "What are source generators? How do they compare to reflection?"
- "Why does Native AOT restrict reflection?"

---

## 11. Going deeper

- [Microsoft docs: Reflection](https://learn.microsoft.com/dotnet/fundamentals/reflection/overview)
- [Microsoft docs: Source generators](https://learn.microsoft.com/dotnet/csharp/roslyn-sdk/source-generators-overview)
- [Microsoft docs: JSON source generation](https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/source-generation)
- [Microsoft docs: Prepare libraries for trimming](https://learn.microsoft.com/dotnet/core/deploying/trimming/prepare-libraries-for-trimming)

**Next:** [Chapter 13 — Dependency Injection](13-dependency-injection.md) covers the
pattern that wires every modern .NET application together.
