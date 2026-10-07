# Dependency Injection

Every ASP.NET Core application starts with a list of `builder.Services.AddSomething()`
calls, and every controller or endpoint receives its dependencies through its
constructor or parameters. Dependency injection (DI) is so built in that it's easy to use
it mechanically: add an interface, register it, inject it. But DI is a design technique
first and a container second. Used well, it makes code testable and flexible. Used
mechanically, it produces hundreds of pointless interfaces, confusing lifetimes and a
startup that crashes with "Cannot consume scoped service from singleton."

---

## 1. The problem: who creates the objects?

Look at a class that creates its own dependencies:

```csharp
public sealed class TicketService
{
    private readonly PostgresTicketRepository _repo = new("Host=prod-db;...");
    private readonly SmtpNotifier _notifier = new("smtp.example.com", 587);

    public async Task AssignAsync(TicketId id, string assignee) { /* ... */ }
}
```

Problems:

1. **It can't be tested** without a real database and mail server.
2. **It can't be reconfigured**: the connection string and SMTP host are baked in.
3. **It decides lifetimes**: each `TicketService` creates its own repository and notifier,
   even if they should be shared or pooled.
4. **It's coupled to concrete implementations**: switching to Slack notifications means
   editing this class.

The fix is a simple idea called **Inversion of Control**: a class shouldn't *create* what
it needs; it should *receive* it.

```csharp
public sealed class TicketService(ITicketRepository repo, INotifier notifier, TimeProvider clock)
{
    public async Task AssignAsync(TicketId id, string assignee, CancellationToken ct) { /* ... */ }
}
```

Now `TicketService` declares what it needs, and someone else decides what to supply. That
"someone else" is the **composition root**: one place, at application startup, where the
object graph is assembled.

> **🧱 Durable:** Dependency injection is just "pass dependencies in, usually through the
> constructor." A DI *container* is an optional tool that automates the assembly. You can
> do DI without a container (it's called *pure DI*), and the design benefits are the same.

---

## 2. The mental model: a container builds object graphs

A **DI container** is a registry plus a factory:

1. **Registration**: at startup, you tell it which implementation satisfies each service
   type, and for how long instances live.
2. **Resolution**: when something asks for a service, the container looks at the
   implementation's constructor, resolves each parameter recursively, and builds the whole
   graph.

```text
 Request for TicketService
   └─ constructor needs ITicketRepository ──► registered as EfTicketRepository
   │                                            └─ needs BeaconDbContext ──► registered (scoped)
   │                                                 └─ needs DbContextOptions ──► registered
   ├─ constructor needs INotifier ───────────► registered as EmailNotifier
   │                                            └─ needs IOptions<SmtpOptions> ──► configuration
   └─ constructor needs TimeProvider ────────► registered as TimeProvider.System (singleton)
```

.NET's built-in container lives in `Microsoft.Extensions.DependencyInjection` and is used
by ASP.NET Core, worker services and console apps built on the Generic Host:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSingleton(TimeProvider.System);
builder.Services.AddScoped<ITicketRepository, EfTicketRepository>();
builder.Services.AddScoped<TicketService>();
builder.Services.AddTransient<INotifier, EmailNotifier>();

var app = builder.Build();
```

---

## 3. Lifetimes

The most important and most error-prone part of DI.

| Lifetime | One instance per... | Typical use |
|---|---|---|
| **Singleton** | Application | Stateless services, caches, clients designed for reuse (`HttpClient` via factory), `TimeProvider`, configuration |
| **Scoped** | Scope (in ASP.NET Core: one HTTP request) | `DbContext`, unit of work, per-request state (current user) |
| **Transient** | Every resolution | Lightweight, stateless services; things that must not be shared |

### Choosing a lifetime

- **Does it hold per-request state, or wrap something that does (`DbContext`)?** Scoped.
- **Is it stateless or thread-safe, and expensive to create?** Singleton.
- **Is it cheap and stateless?** Either. Transient is the safe default for simple
  services; singleton avoids allocation.

### Captive dependencies

The classic DI bug: a longer-lived service holds a shorter-lived one.

```text
 Singleton CacheWarmer ─────holds────► Scoped BeaconDbContext
 (lives forever)                       (meant to live one request; NOT thread-safe)
```

The scoped `DbContext` is now effectively a singleton, shared by every request
concurrently. Results: data corruption, "A second operation was started on this context
instance" exceptions, and stale data.

**The rule: a service may only depend on services with the same or longer lifetime.**

| Consumer ↓ / Dependency → | Singleton | Scoped | Transient |
|---|---|---|---|
| Singleton | ✓ | ✗ captive | ⚠ captive (lives as long as the singleton) |
| Scoped | ✓ | ✓ | ✓ |
| Transient | ✓ | ✓ | ✓ |

ASP.NET Core enables **scope validation** in the Development environment, which throws at
startup ("Cannot consume scoped service 'X' from singleton 'Y'") if you break the rule.
Enable it everywhere:

```csharp
builder.Host.UseDefaultServiceProvider(o =>
{
    o.ValidateScopes = true;
    o.ValidateOnBuild = true;    // also checks that every registration can be constructed
});
```

### When a singleton needs scoped work

A background service (a singleton) that needs a `DbContext` must **create a scope**:

```csharp
public sealed class SlaMonitor(IServiceScopeFactory scopes, ILogger<SlaMonitor> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var timer = new PeriodicTimer(TimeSpan.FromMinutes(1));
        while (await timer.WaitForNextTickAsync(ct))
        {
            await using var scope = scopes.CreateAsyncScope();
            var service = scope.ServiceProvider.GetRequiredService<TicketService>();
            await service.FlagBreachesAsync(ct);
        }   // scope disposed: DbContext disposed
    }
}
```

### Disposal

The container disposes `IDisposable`/`IAsyncDisposable` services it created, when their
scope ends (scoped and transient) or when the app shuts down (singletons). Don't dispose
injected services yourself; you don't own them.

> **⚠️ What can go wrong:** Transient disposable services resolved from the *root*
> provider are tracked until the application shuts down, which is a memory leak if you
> resolve them repeatedly. Resolve from a scope.

---

## 4. Registration patterns

```csharp
// Interface → implementation
services.AddScoped<ITicketRepository, EfTicketRepository>();

// Concrete type (no interface needed)
services.AddScoped<TicketService>();

// Existing instance
services.AddSingleton(TimeProvider.System);

// Factory for complex construction
services.AddSingleton<INotifier>(sp =>
{
    var opts = sp.GetRequiredService<IOptions<NotificationOptions>>().Value;
    return opts.Channel == "slack"
        ? new SlackNotifier(opts.SlackWebhook)
        : new EmailNotifier(opts.Smtp);
});

// Multiple implementations: inject IEnumerable<T> to get them all
services.AddScoped<ITicketRule, TitleMustBeShortRule>();
services.AddScoped<ITicketRule, NoProfanityRule>();
// public sealed class TicketValidator(IEnumerable<ITicketRule> rules)

// Keyed services (.NET 8+): choose an implementation by key
services.AddKeyedSingleton<INotifier, EmailNotifier>("email");
services.AddKeyedSingleton<INotifier, SlackNotifier>("slack");
// public sealed class Escalator([FromKeyedServices("slack")] INotifier urgent)

// Decorator: wrap an implementation (manually, or with a library like Scrutor)
services.AddScoped<EfTicketRepository>();
services.AddScoped<ITicketRepository>(sp =>
    new CachedTicketRepository(sp.GetRequiredService<EfTicketRepository>(),
                               sp.GetRequiredService<IMemoryCache>()));
```

### Organize registrations by feature

Rather than one 300-line `Program.cs`, group registrations in extension methods owned by
each module:

```csharp
public static class TicketsModule
{
    public static IServiceCollection AddTickets(this IServiceCollection services)
    {
        services.AddScoped<TicketService>();
        services.AddScoped<ITicketRepository, EfTicketRepository>();
        return services;
    }
}

builder.Services.AddTickets();
```

---

## 5. Anti-patterns

### Service locator

```csharp
public sealed class TicketService(IServiceProvider services)
{
    public Task AssignAsync(...)
    {
        var repo = services.GetRequiredService<ITicketRepository>();   // ✗ hidden dependency
        ...
    }
}
```

Injecting `IServiceProvider` and pulling dependencies out of it hides what the class
needs, defeats compile-time checking and makes tests guess what to register. Use it only
in infrastructure code that genuinely must resolve dynamically (factories, the scope
pattern above).

### Constructor over-injection

A constructor with eight dependencies is not a DI problem; it's a design signal. The class
probably has too many responsibilities. Split it.

### Interfaces for everything

Chapter 3 covered this: an interface per class "for DI" isn't required. The built-in
container injects concrete classes perfectly well. Add interfaces for real substitution
needs: I/O boundaries, multiple implementations, module boundaries.

### Injecting configuration as primitives

```csharp
public EmailNotifier(string smtpHost, int port)   // ✗ the container can't resolve "string"
public EmailNotifier(IOptions<SmtpOptions> options)   // ✓ the options pattern (Book III, Chapter 3)
```

### Static access

`DateTime.Now`, `File.ReadAllText`, `Environment.GetEnvironmentVariable` and static
singletons are hidden dependencies too. Inject `TimeProvider`, an abstraction over the
file system where it matters, and options for configuration.

---

## 6. DI and testing

DI's biggest practical payoff is testability. With dependencies passed in, a test can
supply fakes:

```csharp
var clock = new FakeTimeProvider(new DateTimeOffset(2026, 10, 1, 9, 0, 0, TimeSpan.Zero));
var repo = new InMemoryTicketRepository();
var notifier = new RecordingNotifier();
var service = new TicketService(repo, notifier, clock);
```

No container involved; tests just call constructors. (`FakeTimeProvider` is in the
`Microsoft.Extensions.TimeProvider.Testing` package.) Chapter 14 builds on this.

> **🧭 When not to use a container:** Small console tools, libraries, and code with only a
> handful of objects can wire dependencies by hand in `Main`. A container earns its place
> when the graph is large, lifetimes matter (per-request scopes), or a framework expects
> one. Libraries should never require a specific container; they expose constructor
> parameters and, optionally, an `AddMyLibrary()` extension.

---

## 7. In practice: Beacon's composition root

The CLI has been creating objects by hand. Let's move it onto the Generic Host so it uses
the same DI, configuration and logging as the API will in Book III.

```bash
dotnet add src/Beacon.Cli package Microsoft.Extensions.Hosting
```

First, Beacon.Core exposes a registration method for its own services, keeping the
knowledge of what Core needs inside Core:

```csharp
// src/Beacon.Core/DependencyInjection.cs
using Beacon.Core.Auditing;
using Beacon.Core.Common;
using Beacon.Core.Indexing;
using Beacon.Core.Tickets;
using Microsoft.Extensions.DependencyInjection;

namespace Beacon.Core;

public static class DependencyInjection
{
    public static IServiceCollection AddBeaconCore(this IServiceCollection services)
    {
        services.AddSingleton(TimeProvider.System);
        services.AddSingleton<AuditLog>();
        services.AddSingleton<IndexingQueue>();
        services.AddSingleton<EventDispatcher>();
        services.AddScoped<TicketService>();
        return services;
    }
}
```

(Add the `Microsoft.Extensions.DependencyInjection.Abstractions` package to Beacon.Core.
It's just the interfaces, not the container.)

Then the CLI's composition root supplies the infrastructure:

```csharp
// src/Beacon.Cli/Program.cs
using Beacon.Cli;
using Beacon.Core;
using Beacon.Core.Notifications;
using Beacon.Core.Tickets;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.Hosting;

var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddBeaconCore();
builder.Services.AddSingleton<ITicketRepository>(new InMemoryTicketRepository(TimeSpan.FromMilliseconds(20)));
builder.Services.AddSingleton<INotifier, ConsoleNotifier>();
builder.Services.AddSingleton<TicketIdGenerator>();
builder.Services.AddTransient<CliApp>();

builder.Services.Configure<HostOptions>(_ => { });   // placeholder for later configuration
using var host = builder.Build();

await using var scope = host.Services.CreateAsyncScope();
var app = scope.ServiceProvider.GetRequiredService<CliApp>();
return await app.RunAsync(args, CancellationToken.None);
```

```csharp
// src/Beacon.Cli/CliApp.cs
using Beacon.Core.Tickets;

namespace Beacon.Cli;

internal sealed class CliApp(TicketService tickets, ITicketRepository repo, TicketIdGenerator ids, TimeProvider clock)
{
    public async Task<int> RunAsync(string[] args, CancellationToken ct)
    {
        var ticket = new Ticket(ids.Next(), "Printer on 3rd floor offline", TicketPriority.Normal, clock.GetUtcNow());
        await repo.SaveAsync(ticket, ct);

        var result = await tickets.AddCommentAsync(ticket.Id, "maria", "Power-cycled it; investigating.", ct);
        Console.WriteLine(result.Match(t => $"{t.Id} now has {t.Comments.Count} comment(s).", e => e.Message));
        return result.IsSuccess ? 0 : 1;
    }
}
```

Note the lifetimes:

- `InMemoryTicketRepository` is a **singleton** because it *is* the data store in this
  CLI and is thread-safe (`ConcurrentDictionary`). When it becomes an EF Core repository,
  it will become **scoped**, and the composition root is the only place that changes.
- `TicketService` is **scoped**, so we create a scope to resolve it, exactly as ASP.NET
  Core will do per request.
- The registration for `ITicketRepository` uses an instance, so the container doesn't
  own its disposal; that's fine for an object that lives for the whole process anyway.

---

## 8. What can go wrong

- **Captive dependencies**: scoped or transient services trapped in singletons.
- **Missing registrations**, discovered only when a rarely used endpoint is hit. Fix with
  `ValidateOnBuild`.
- **Service locator** usage hiding dependencies.
- **Disposing injected services** you don't own.
- **Resolving scoped services from the root provider** (outside any scope).
- **Huge constructors** signaling classes that do too much.
- **Registration order surprises**: when multiple registrations exist for the same
  service, resolving a single instance returns the *last* one registered.
- **Thread-safety of singletons** (Chapter 10).

---

## 9. How an experienced engineer thinks about this

- **DI is a design principle; the container is a convenience.** Constructor parameters
  are the API; the container just calls them.
- **Lifetimes are a correctness issue, not a performance tweak.** Decide them
  deliberately; validate scopes.
- **Keep the composition root at the edge.** Domain code never references the container.
- **Interfaces at boundaries, concrete classes inside.**
- **Explicit registration** that a reader can follow beats clever assembly scanning.

---

## 10. Check yourself

**Questions**

1. What problem does inversion of control solve? Can you do DI without a container?
2. Explain singleton, scoped and transient lifetimes with an example of each.
3. What's a captive dependency, and why is a captive `DbContext` dangerous?
4. How should a `BackgroundService` use a scoped service?
5. What's wrong with the service locator pattern?
6. What does `ValidateOnBuild` check?

**Exercises**

1. Register a scoped service inside a singleton in a small ASP.NET Core app with scope
   validation on, and read the error. Then fix it with `IServiceScopeFactory`.
2. Implement a `CachedTicketRepository` decorator and register it so `TicketService`
   gets the cached version without changing its code.
3. Use keyed services to register two `INotifier`s and inject the right one into two
   different consumers.
4. Rewrite the CLI's composition root using *pure DI* (no container). Compare the two.

**Interview-style questions**

- "Explain dependency injection and why it's useful."
- "What are the service lifetimes in ASP.NET Core? Give a bug caused by choosing wrong."
- "What's the difference between DI and the service locator pattern?"

---

## 11. Going deeper

- [Microsoft docs: Dependency injection in .NET](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection)
- [Microsoft docs: DI guidelines](https://learn.microsoft.com/dotnet/core/extensions/dependency-injection-guidelines)
- Mark Seemann & Steven van Deursen, *Dependency Injection: Principles, Practices, and
  Patterns* — the definitive book on the topic.

**Next:** [Chapter 14 — Testing](14-testing.md) uses everything in Book I to build a test
suite you'll actually want to keep.
