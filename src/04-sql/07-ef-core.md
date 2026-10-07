# EF Core

> **🔄 Current (as of October 2026):** EF Core 10 ships with .NET 10 (LTS), with the Npgsql
> provider `Npgsql.EntityFrameworkCore.PostgreSQL` 10.x. Recent additions referenced here:
> `ExecuteUpdate`/`ExecuteDelete` (EF7+), complex types (EF8+, extended in 9 and 10),
> named query filters and the `LeftJoin` LINQ operator (EF10).

Entity Framework Core is .NET's standard ORM (object-relational mapper). It maps classes
to tables, translates LINQ into SQL, tracks changes to loaded objects, and generates
migrations. Used well, it removes a large amount of tedious data-access code. Used
carelessly, it's the source of the N+1 queries, accidental full-table loads and
mysterious performance problems from Chapters 4–6.

This chapter treats EF Core as what it is: a SQL generator with a change tracker. Knowing
the SQL it produces, and when to bypass it, is the difference between the two outcomes.

---

## 1. The problem: objects and tables don't match

C# has objects with references, inheritance, collections and encapsulation. Relational
databases have tables, rows, foreign keys and sets. Bridging them (the
**object-relational impedance mismatch**) requires answering:

- How does a class map to a table, and a property to a column?
- How do object references become foreign keys and joins?
- When is related data loaded?
- How do changes to objects become `insert`, `update` and `delete` statements?
- How does the schema evolve with the code?

An ORM automates those answers. The price is an abstraction that can hide what's happening
in the database. The cure is to look at the SQL.

---

## 2. The mental model

```text
 Your code ──LINQ──► DbContext ──► Query pipeline: expression tree → SQL ──► Npgsql ──► PostgreSQL
                       │                                                     ◄── rows
                       │◄── materialize entities ◄──────────────────────────────┘
                       │
                       └─ Change tracker: snapshots of loaded entities
                          SaveChanges(): compare → INSERT/UPDATE/DELETE in a transaction
```

- **`DbContext`**: a *unit of work* plus a gateway to the database. It's lightweight,
  **not thread-safe**, and meant to be short-lived: in ASP.NET Core, one per request
  (scoped; Book I, Chapter 13).
- **`DbSet<T>`**: an `IQueryable<T>` over a table. LINQ on it is translated to SQL via
  expression trees (Book I, Chapters 6 and 7).
- **Change tracker**: remembers entities loaded by the context and their original values.
  `SaveChanges` detects what changed and generates the SQL.
- **Model**: built from your classes, conventions and configuration; it describes every
  table, column, key and relationship.

---

## 3. Mapping Beacon's domain

### The DbContext

```csharp
// src/Beacon.Infrastructure/Persistence/BeaconDbContext.cs
using Beacon.Core.Tickets;
using Microsoft.EntityFrameworkCore;

namespace Beacon.Infrastructure.Persistence;

public sealed class BeaconDbContext(DbContextOptions<BeaconDbContext> options) : DbContext(options)
{
    public DbSet<Ticket> Tickets => Set<Ticket>();
    public DbSet<OutboxMessage> Outbox => Set<OutboxMessage>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
        => modelBuilder.ApplyConfigurationsFromAssembly(typeof(BeaconDbContext).Assembly);
}
```

A new project holds infrastructure, keeping `Beacon.Core` free of EF Core (the
architecture test from Book I, Chapter 14 enforces this):

```bash
dotnet new classlib -n Beacon.Infrastructure -o src/Beacon.Infrastructure
dotnet sln add src/Beacon.Infrastructure
dotnet add src/Beacon.Infrastructure reference src/Beacon.Core
dotnet add src/Beacon.Infrastructure package Npgsql.EntityFrameworkCore.PostgreSQL
dotnet add src/Beacon.Api reference src/Beacon.Infrastructure
```

### Mapping a rich domain entity

Beacon's `Ticket` has private setters, a private comment list, a strongly typed ID and
domain events (Book I). EF Core can map all of that without compromising the domain:

```csharp
// src/Beacon.Infrastructure/Persistence/TicketConfiguration.cs
using Beacon.Core.Tickets;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;

namespace Beacon.Infrastructure.Persistence;

internal sealed class TicketConfiguration : IEntityTypeConfiguration<Ticket>
{
    public void Configure(EntityTypeBuilder<Ticket> b)
    {
        b.ToTable("tickets");
        b.HasKey(t => t.Id);
        b.Property(t => t.Id)
            .HasConversion(id => id.Value, value => new TicketId(value))    // TicketId ⇄ bigint
            .UseIdentityAlwaysColumn();

        b.Property(t => t.Title).HasMaxLength(200).IsRequired();
        b.Property(t => t.Description).HasMaxLength(10_000);
        b.Property(t => t.Priority).HasConversion<short>();
        b.Property(t => t.Status).HasConversion<short>();
        b.Property(t => t.CreatedAt);
        b.Property(t => t.ReporterId).IsRequired();
        b.Property(t => t.Version).IsConcurrencyToken();                   // optimistic concurrency

        b.Ignore(t => t.DomainEvents);                                      // not persisted
        b.Ignore(t => t.IsActive);                                          // computed in C#

        b.HasMany(t => t.Comments)
         .WithOne()
         .HasForeignKey("TicketId")                                         // shadow property
         .OnDelete(DeleteBehavior.Cascade);
        b.Navigation(t => t.Comments).UsePropertyAccessMode(PropertyAccessMode.Field);   // uses _comments

        b.HasIndex(t => new { t.TeamId, t.CreatedAt, t.Id })
         .IsDescending(false, true, true)
         .HasFilter("status in (0, 1)")
         .HasDatabaseName("tickets_team_open_page");
    }
}
```

Points worth noticing:

- **Value converters** map `TicketId` to `bigint` and enums to `smallint`.
- **Private setters and backing fields**: EF Core sets properties via their private setters
  or fields; it doesn't need public setters. The domain stays encapsulated.
- **Shadow properties** (`TicketId` on comments) exist in the model and database but not in
  the C# class: the domain's `Comment` record doesn't need to know its ticket's ID.
- `Comment` needs its own key; a small configuration gives it a shadow `Id` identity column.
- **`IsConcurrencyToken`** on `Version` makes every update include `where version = @original`
  (Chapter 5's optimistic concurrency), throwing `DbUpdateConcurrencyException` on conflict.
  The domain increments `Version` in each mutating method.
- **Indexes** from Chapter 4 are declared here, so migrations create them.

Use **snake_case** naming to match PostgreSQL conventions, with the `EFCore.NamingConventions`
package: `options.UseSnakeCaseNamingConvention()`.

### Registration

```csharp
// src/Beacon.Infrastructure/DependencyInjection.cs
public static IServiceCollection AddBeaconInfrastructure(this IServiceCollection services, IConfiguration config)
{
    services.AddDbContext<BeaconDbContext>(o => o
        .UseNpgsql(config.GetConnectionString("Beacon"), npgsql => npgsql.EnableRetryOnFailure())
        .UseSnakeCaseNamingConvention());

    services.AddScoped<ITicketRepository, EfTicketRepository>();
    return services;
}
```

`EnableRetryOnFailure` retries transient connection failures. (It requires wrapping
user-initiated transactions in an execution strategy; see section 7.)

---

## 4. Querying

### See the SQL

Before anything else, make SQL visible in development:

```json
"Logging": { "LogLevel": { "Microsoft.EntityFrameworkCore.Database.Command": "Information" } }
```

Or per query: `var sql = query.ToQueryString();`.

### Tracking vs no-tracking

By default, queried entities are **tracked**: snapshotted so `SaveChanges` can detect
changes. Tracking costs memory and CPU. For read-only queries, turn it off:

```csharp
var tickets = await db.Tickets.AsNoTracking().Where(...).ToListAsync(ct);
```

### Projection: fetch only what you need

The most effective EF Core performance technique:

```csharp
// ✗ Loads full entities (all columns), tracked, then maps in memory
var list = await db.Tickets.Where(t => t.TeamId == teamId).ToListAsync(ct);
var dtos = list.Select(TicketListItem.From);

// ✓ SQL selects only these columns; no tracking; comment count computed in SQL
var dtos = await db.Tickets
    .Where(t => t.TeamId == teamId && (t.Status == TicketStatus.Open || t.Status == TicketStatus.InProgress))
    .OrderByDescending(t => t.CreatedAt).ThenByDescending(t => t.Id)
    .Select(t => new TicketListItem(t.Id, t.Title, t.Status, t.Priority, t.AssigneeId, t.CreatedAt, t.Comments.Count))
    .Take(21)
    .ToListAsync(ct);
```

Generated SQL (simplified):

```sql
select t.id, t.title, t.status, t.priority, t.assignee_id, t.created_at,
       (select count(*)::int from comments c where t.id = c.ticket_id)
from tickets t
where t.team_id = @teamId and t.status in (0, 1)
order by t.created_at desc, t.id desc
limit 21
```

One query, exactly the needed columns, served by `tickets_team_open_page`.

### Loading related data

| Strategy | How | Risk |
|---|---|---|
| **Eager** | `Include(t => t.Comments)` | Cartesian explosion with multiple collections |
| **Split query** | `Include(...).AsSplitQuery()` | One query per collection; consistency between queries |
| **Explicit** | `await db.Entry(t).Collection(x => x.Comments).LoadAsync()` | Extra round trip |
| **Lazy loading** | Navigation loads on first access (proxies) | **N+1** silently, everywhere |
| **Projection** | `Select(t => new { ..., Comments = t.Comments.Select(...) })` | Usually the best for reads |

> **🧭 When not to use lazy loading:** Almost never enable it in web applications. It
> turns innocent loops (`foreach (var t in tickets) Console.Write(t.Comments.Count)`) into
> one query per iteration: the N+1 problem, hidden in property access. Load explicitly.

### The N+1 problem in EF Core

```csharp
var tickets = await db.Tickets.Where(...).ToListAsync(ct);   // 1 query
foreach (var t in tickets)
    t.LastComment = await db.Comments.Where(c => c.TicketId == t.Id)   // N queries
        .OrderByDescending(c => c.CreatedAt).FirstOrDefaultAsync(ct);
```

Fix with one projected query (EF translates the subquery to a lateral join or window
function):

```csharp
var rows = await db.Tickets.Where(...)
    .Select(t => new
    {
        t.Id, t.Title,
        LastComment = t.Comments.OrderByDescending(c => c.CreatedAt).Select(c => c.Body).FirstOrDefault()
    })
    .ToListAsync(ct);
```

### Translation limits

Only expressions EF Core understands become SQL. `t.IsActive` (a C# property) and
`SlaRules.IsBreaching(t, now)` (a C# method) can't be translated: EF Core throws
*"could not be translated."* Options:

- Express the condition with mapped columns: `t.Status == Open || t.Status == InProgress`.
- Map a computed column in the database (`is_open` generated column, Chapter 3).
- Store the SLA targets in a table and join (keeps one source of truth: Chapter 2's tension).
- Narrow in SQL, then finish in memory with `AsEnumerable()` deliberately (and only on a
  small, already-filtered set).

### Raw SQL when it's clearer

For reporting queries with window functions and CTEs (Chapter 2), write SQL:

```csharp
var report = await db.Database
    .SqlQuery<AgentWorkloadRow>($"""
        select u.display_name as agent, count(*) as active, min(t.created_at) as oldest_active
        from tickets t join users u on u.id = t.assignee_id
        where t.status in (0, 1) and t.team_id = {teamId}
        group by u.display_name
        order by active desc
        """)
    .ToListAsync(ct);
```

`SqlQuery` with an interpolated string **parameterizes** `{teamId}` (Book III, Chapter 9).
Dapper is another excellent option for query-heavy read paths; many teams use EF Core
for writes and Dapper or raw SQL for complex reads.

---

## 5. Saving

```csharp
var ticket = await db.Tickets.Include(t => t.Comments).SingleOrDefaultAsync(t => t.Id == id, ct);
ticket!.AddComment(new Comment(user.Id, body, clock.GetUtcNow()));
await db.SaveChangesAsync(ct);
```

`SaveChanges`:

1. Runs `DetectChanges`: compares tracked entities with their snapshots.
2. Opens a transaction (if none exists).
3. Sends `insert`/`update`/`delete` statements (batched).
4. Commits, then updates snapshots and generated values (IDs).

### Bulk operations without loading

Loading entities to change them one by one is wasteful for set-based changes (Chapter 2).
`ExecuteUpdate` and `ExecuteDelete` translate directly to SQL:

```csharp
await db.Tickets
    .Where(t => t.Status == TicketStatus.Resolved && t.ResolvedAt < now.AddDays(-7))
    .ExecuteUpdateAsync(s => s
        .SetProperty(t => t.Status, TicketStatus.Closed)
        .SetProperty(t => t.Version, t => t.Version + 1), ct);
```

They bypass the change tracker and domain logic (no domain events, no invariants checked
in C#), so use them deliberately for operations where that's acceptable.

---

## 6. Migrations

EF Core migrations generate schema changes from model changes:

```bash
dotnet tool install -g dotnet-ef
dotnet ef migrations add InitialSchema -p src/Beacon.Infrastructure -s src/Beacon.Api
dotnet ef migrations script --idempotent -o migrations.sql -p src/Beacon.Infrastructure -s src/Beacon.Api
```

### Rules for production migrations

- **Review the generated migration and SQL.** Generators can't know intent: renaming a
  property may be generated as *drop column + add column*, deleting data.
- **Apply migrations as a separate deployment step** (a SQL script or a migration bundle:
  `dotnet ef migrations bundle`), run by the `beacon_migrator` role (Chapter 3), not by the
  application at startup. `Database.Migrate()` at startup is fine for development, but in
  production it races between instances and requires the app to have DDL privileges.
- **Make migrations backward compatible** so old and new versions of the app can run
  during a rolling deployment (**expand and contract**):

```text
 Rename column "assignee" → "assignee_id":
 1. Expand:   add assignee_id; app writes both, reads assignee_id with fallback   (deploy)
 2. Migrate:  backfill assignee_id from assignee in batches
 3. Contract: app stops using assignee                                             (deploy)
 4. Drop:     drop column assignee                                                 (later)
```

- **Mind locks** (Chapter 5): set `lock_timeout`, create indexes `concurrently` (EF Core
  migrations run in a transaction by default; put concurrent index creation in its own
  migration with `migrationBuilder.Sql("create index concurrently ...", suppressTransaction: true)`).
- **Large backfills in batches**, not one huge `update` that locks the table and bloats the WAL.

---

## 7. Transactions and the outbox in EF Core

`SaveChanges` is already transactional. For several `SaveChanges` calls or raw SQL in one
unit, use an explicit transaction, inside the execution strategy when retries are enabled:

```csharp
var strategy = db.Database.CreateExecutionStrategy();
await strategy.ExecuteAsync(async () =>
{
    await using var tx = await db.Database.BeginTransactionAsync(ct);
    // ... work ...
    await db.SaveChangesAsync(ct);
    await tx.CommitAsync(ct);
});
```

### Domain events → outbox, automatically

Chapter 5 designed the outbox table. With EF Core, a `SaveChanges` interceptor converts
domain events into outbox rows in the same transaction, so no business code can forget:

```csharp
// src/Beacon.Infrastructure/Persistence/OutboxMessage.cs
public sealed class OutboxMessage
{
    public long Id { get; init; }
    public required string Type { get; init; }
    public required string Payload { get; init; }      // jsonb
    public DateTimeOffset CreatedAt { get; init; }
    public DateTimeOffset? ProcessedAt { get; set; }
    public int Attempts { get; set; }
    public string? LastError { get; set; }
}
```

```csharp
// src/Beacon.Infrastructure/Persistence/DomainEventsToOutboxInterceptor.cs
using System.Text.Json;
using Beacon.Core.Common;
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Diagnostics;

namespace Beacon.Infrastructure.Persistence;

public sealed class DomainEventsToOutboxInterceptor(TimeProvider clock) : SaveChangesInterceptor
{
    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, InterceptionResult<int> result, CancellationToken ct = default)
    {
        var db = eventData.Context!;
        var entities = db.ChangeTracker.Entries()
            .Select(e => e.Entity)
            .OfType<IHasDomainEvents>()
            .Where(e => e.DomainEvents.Count > 0)
            .ToList();

        foreach (var entity in entities)
        {
            foreach (var evt in entity.DomainEvents)
            {
                db.Set<OutboxMessage>().Add(new OutboxMessage
                {
                    Type = evt.GetType().Name,
                    Payload = JsonSerializer.Serialize(evt, evt.GetType()),
                    CreatedAt = clock.GetUtcNow(),
                });
            }
            entity.ClearDomainEvents();
        }

        return base.SavingChangesAsync(eventData, result, ct);
    }
}
```

(`IHasDomainEvents` is a small interface in `Beacon.Core.Common` that `Ticket` implements.)
Register it with `o.AddInterceptors(sp.GetRequiredService<DomainEventsToOutboxInterceptor>())`.
The outbox worker (Chapter 5's `skip locked` query) publishes the messages to the
in-process dispatcher, emails and SignalR (Book III, Chapter 8).

---

## 8. The repository question

Should you wrap EF Core in repositories? Book I, Chapter 4 warned against *generic*
repositories over `DbContext`. Beacon uses **specific** ones for aggregates with domain
behavior, and queries `DbContext` directly (or with raw SQL) for read models:

```csharp
// src/Beacon.Infrastructure/Persistence/EfTicketRepository.cs
public sealed class EfTicketRepository(BeaconDbContext db) : ITicketRepository
{
    public Task<Ticket?> FindAsync(TicketId id, CancellationToken ct = default)
        => db.Tickets.Include(t => t.Comments).SingleOrDefaultAsync(t => t.Id == id, ct);

    public async Task SaveAsync(Ticket ticket, CancellationToken ct = default)
    {
        if (db.Entry(ticket).State == EntityState.Detached) db.Tickets.Add(ticket);
        await db.SaveChangesAsync(ct);
    }
}
```

The domain and application services (`TicketService`) depend on `ITicketRepository`, so
they stay testable with fakes (Book I, Chapter 14). Read-heavy endpoints (lists, search,
reports) query `BeaconDbContext` with projections, because they don't need domain objects.
This split between **commands through the domain** and **queries straight to read models**
is a lightweight form of CQRS (Book XIII).

---

## 9. In practice: switching Beacon to PostgreSQL

1. **Infrastructure project** with `BeaconDbContext`, configurations, `EfTicketRepository`,
   the outbox interceptor and an outbox worker.
2. **Remove `InMemoryTicketRepository`** from Beacon.Api; call `AddBeaconInfrastructure`.
3. **Initial migration** generating the schema from Chapter 1 (compare the generated SQL to
   the hand-written schema and adjust configuration until they match).
4. **The list endpoint** becomes the projected query from section 4. The cursor condition is
   the one place LINQ gets awkward: EF Core can't translate `t.Id.Value` on a value-converted
   property, and C# has no row-comparison syntax. EF Core lets you start from SQL and keep
   composing in LINQ, which handles it neatly:

```csharp
IQueryable<Ticket> source = cursor is null
    ? db.Tickets
    : db.Tickets.FromSql($"select * from tickets where (created_at, id) < ({cursor.Value.CreatedAt}, {cursor.Value.Id})");

var items = await source
    .Where(t => t.TeamId == teamId && (t.Status == TicketStatus.Open || t.Status == TicketStatus.InProgress))
    .OrderByDescending(t => t.CreatedAt).ThenByDescending(t => t.Id)
    .Select(t => new TicketListItem(t.Id, t.Title, t.Status, t.Priority, t.AssigneeId, t.CreatedAt, t.Comments.Count))
    .Take(limit + 1)
    .ToListAsync(ct);
```

   The interpolated values are parameterized, and PostgreSQL's row comparison
   `(created_at, id) < (@p0, @p1)` matches the `tickets_team_open_page` index exactly.

5. **Integration tests against real PostgreSQL** with Testcontainers:

```csharp
// tests/Beacon.Api.Tests/BeaconApiFactory.cs
using Microsoft.AspNetCore.Mvc.Testing;
using Testcontainers.PostgreSql;

public sealed class BeaconApiFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _db = new PostgreSqlBuilder().WithImage("postgres:18").Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
        => builder.UseSetting("ConnectionStrings:Beacon", _db.GetConnectionString());

    public async ValueTask InitializeAsync()
    {
        await _db.StartAsync();
        using var scope = Services.CreateScope();
        await scope.ServiceProvider.GetRequiredService<BeaconDbContext>().Database.MigrateAsync();
    }

    public override async ValueTask DisposeAsync() { await _db.DisposeAsync(); await base.DisposeAsync(); }
}
```

   Now the tests exercise real SQL, real constraints, real migrations. Add tests that assert
   the number of SQL commands per request for list endpoints (an interceptor that counts
   `DbCommand` executions), to catch N+1 regressions (Book III, Chapter 10's postmortem
   action item).

6. **Book I's LINQ report** (`AgentWorkloadReport`) now runs against an `IQueryable`
   and fails to translate `IsActive` and `SlaRules.IsBreaching`, exactly as predicted. It
   becomes the raw SQL query from Chapter 2, reading SLA targets from a small `sla_targets`
   table that both C# and SQL use.

---

## 10. What can go wrong

- **N+1** from lazy loading or queries in loops.
- **Loading whole entities** (and tracking them) for read-only lists.
- **Cartesian explosion** with multiple `Include`s of collections.
- **Client evaluation surprises**: an `AsEnumerable()` or C# method silently moving
  filtering into memory.
- **Long-lived or shared `DbContext`** (singleton, static, across threads): stale data,
  memory growth, concurrency exceptions.
- **Migrations that drop data** (renames as drop + add), lock tables, or break the old app
  version during rolling deploys.
- **`Database.Migrate()` at startup** in multi-instance production.
- **Bulk operations through the change tracker**: loading 100,000 entities to update one column.
- **Not looking at the generated SQL.**

---

## 11. How an experienced engineer thinks about this

- **EF Core is a SQL generator; always know the SQL.**
- **Commands through the domain, queries through projections.** Track only what you'll
  change.
- **No lazy loading; explicit loading strategies.**
- **Drop to SQL when it's clearer**, especially for reporting.
- **Migrations are production changes**: reviewed, backward compatible, run deliberately.
- **Test against the real database engine.**

---

## 12. Check yourself

**Questions**

1. What does the change tracker do, and when should you turn tracking off?
2. Why is projection with `Select` usually better than loading entities for reads?
3. What's the N+1 problem, and how does lazy loading make it worse?
4. How does EF Core implement optimistic concurrency?
5. What do `ExecuteUpdate`/`ExecuteDelete` skip?
6. Why shouldn't production apps run `Database.Migrate()` at startup?
7. Explain expand-and-contract migrations.
8. How does a `SaveChanges` interceptor help implement the outbox?

**Exercises**

1. Map `Article` with its `search` generated column and implement the search endpoint
   using `EF.Functions.WebSearchToTsQuery` and `Matches` (Npgsql's full-text support).
2. Write a test that counts SQL commands for `GET /api/tickets` and fails above 2.
3. Simulate a concurrent update of the same ticket in two contexts and handle
   `DbUpdateConcurrencyException` by returning `409`/`412`.
4. Rename `Ticket.AssigneeId` to `AssignedUserId` with an expand-and-contract migration plan.

**Interview-style questions**

- "What are the pros and cons of using an ORM?"
- "How do you avoid N+1 queries in Entity Framework?"
- "How do you handle database migrations in a continuously deployed system?"
- "Repository pattern over EF Core: yes or no?"

---

## 13. Going deeper

- [Microsoft docs: EF Core](https://learn.microsoft.com/ef/core/) — especially "Performance."
- [Npgsql EF Core provider docs](https://www.npgsql.org/efcore/)
- [Testcontainers for .NET](https://dotnet.testcontainers.org/)
- Jon P Smith, *Entity Framework Core in Action*.

**Next:** [Chapter 8 — Database Security and Operations](08-database-security-and-operations.md)
covers keeping the database safe, backed up and healthy in production.
