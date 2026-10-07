# ASP.NET Core Fundamentals

> **🔄 Current (as of October 2026):** This chapter targets ASP.NET Core 10. Notable
> recent additions used in this book: built-in OpenAPI document generation
> (`Microsoft.AspNetCore.OpenApi`, replacing Swashbuckle in templates since .NET 9),
> built-in validation for minimal APIs (.NET 10), `TypedResults.ServerSentEvents`
> (.NET 10), and `IExceptionHandler` (.NET 8).

ASP.NET Core is how Beacon becomes a web application. It's a fast, modular framework, but
"modular" means a lot happens between a request arriving and your code running: a server
accepts the connection, a pipeline of middleware processes the request, routing picks an
endpoint, parameters are bound, and filters run. When something behaves unexpectedly (a
route doesn't match, authentication doesn't apply, an exception produces an HTML page
instead of JSON), knowing that pipeline is how you find out why.

---

## 1. The problem: from bytes on a socket to your method

Between the network and `TicketService.AddCommentAsync`, a web framework must:

1. Accept TCP connections, handle TLS, and parse HTTP (any version).
2. Run cross-cutting logic for every request: logging, error handling, HTTPS redirection,
   authentication, CORS, compression, rate limiting.
3. Decide which code handles this URL and method (routing).
4. Convert route values, query strings, headers and JSON bodies into typed parameters
   (binding), and validate them.
5. Call your code with its dependencies.
6. Turn the result into an HTTP response (serialization, status codes).
7. Do all of this concurrently for thousands of requests, efficiently.

ASP.NET Core divides those jobs into well-defined pieces you can see and control.

---

## 2. The mental model

```text
 Network ─► Kestrel (server) ─► HttpContext ─► Middleware pipeline ─────────────────────► Endpoint
                                               │ ExceptionHandler                        (your code)
                                               │ HttpsRedirection                            │
                                               │ Routing (match endpoint)                    │
                                               │ Authentication / Authorization              │
                                               │ RateLimiter, CORS, ...                      │
                                               ▼                                             ▼
 Network ◄─ Kestrel ◄────────────────────────── response flows back through middleware ◄─ result
```

### The host

`WebApplication.CreateBuilder(args)` creates a **host**: the container for configuration
(Chapter 3), logging, DI (Book I, Chapter 13) and the server. `builder.Build()` freezes
the service registrations; `app.Run()` starts listening.

### Kestrel

Kestrel is ASP.NET Core's cross-platform web server, built on async sockets. It's
fast enough to face the internet directly, but in production it's usually behind a
reverse proxy or load balancer (Nginx, Azure Front Door, Kubernetes ingress) that handles
TLS termination, routing to multiple instances, and more (Book VIII).

### `HttpContext`

Each request gets an `HttpContext`: the request (`Method`, `Path`, `Query`, `Headers`,
`Body`), the response, the authenticated `User`, `RequestServices` (the request's DI
scope), `RequestAborted` (a cancellation token for client disconnects), and `Items` for
per-request data.

### Middleware

A **middleware** is a component that receives the `HttpContext`, can do work, and
decides whether to call the **next** middleware:

```csharp
app.Use(async (context, next) =>
{
    var sw = Stopwatch.StartNew();
    await next(context);                                   // run the rest of the pipeline
    context.Response.Headers["X-Elapsed-Ms"] = sw.ElapsedMilliseconds.ToString();  // ✗ too late!
});
```

That last line is a classic bug: by the time `next` returns, the response has usually
started streaming and headers can't be modified. Use `context.Response.OnStarting(...)` to
set headers just before they're sent.

The pipeline is a chain of delegates (`RequestDelegate`), each wrapping the next, like
Russian dolls. **Order matters**:

- Exception handling must be *first*, so it wraps everything.
- `UseAuthentication` must run before `UseAuthorization`.
- CORS must run before anything that might short-circuit with a response.
- Static files often go early, so they skip unnecessary work.

A middleware can **short-circuit** by not calling `next`: rate limiting returning `429`,
authentication challenges, static file responses.

> **🧱 Durable:** The middleware pattern (a pipeline of handlers, each deciding whether to
> pass the request on) is everywhere: Express.js, Go's `net/http`, Python's WSGI/ASGI, HTTP
> client handlers (`DelegatingHandler`), and message processing frameworks. Learn it once.

### Routing and endpoints

**Endpoint routing** separates two steps: `UseRouting` (implicit in minimal hosting)
*matches* the request to an endpoint and stores it on the context; later middleware can
inspect the chosen endpoint (authorization reads its `[Authorize]` metadata); finally the
endpoint executes. Route templates support parameters and constraints:

```text
/api/tickets/{id:int}              id must be an integer
/api/tickets/{id:int:min(1)}       ...and at least 1
/api/files/{**path}                catch-all: the rest of the path
```

---

## 3. Minimal APIs and controllers

ASP.NET Core offers two styles for defining endpoints.

### Minimal APIs

```csharp
app.MapGet("/api/tickets/{id:int}", async (int id, ITicketRepository repo, CancellationToken ct) =>
{
    var ticket = await repo.FindAsync(new TicketId(id), ct);
    return ticket is null ? Results.NotFound() : Results.Ok(TicketSummary.From(ticket));
});
```

Handlers are lambdas or methods; parameters are bound automatically from the route, query,
headers, body and DI.

### Controllers

```csharp
[ApiController]
[Route("api/tickets")]
public sealed class TicketsController(ITicketRepository repo) : ControllerBase
{
    [HttpGet("{id:int}")]
    public async Task<ActionResult<TicketSummary>> Get(int id, CancellationToken ct)
    {
        var ticket = await repo.FindAsync(new TicketId(id), ct);
        return ticket is null ? NotFound() : TicketSummary.From(ticket);
    }
}
```

### Choosing

| | Minimal APIs | Controllers (MVC) |
|---|---|---|
| Ceremony | Low | More (classes, attributes) |
| Performance | Slightly faster | Fast |
| Native AOT | Supported | Not supported |
| Filters, conventions | Endpoint filters, route groups | Rich filter pipeline, conventions, model binding extensibility |
| Organization | Route groups + extension methods | Classes group actions naturally |
| Where Microsoft invests most | Here | Mature, stable |

Both are production-ready. Large existing codebases are mostly controllers. New services,
and this book, use **minimal APIs**, organized with route groups so they don't become a
3,000-line `Program.cs`.

> **🧭 When not to use minimal APIs:** If your team already has a large controller-based
> codebase with custom filters and conventions, consistency is worth more than the small
> benefits of switching. Mixing styles in one app is possible but confusing.

### Typed results

Minimal API handlers can return `TypedResults`, which carry their status code and response
type in the method signature. That makes handlers unit-testable and gives OpenAPI accurate
metadata:

```csharp
static async Task<Results<Ok<TicketSummary>, NotFound>> GetTicket(int id, ITicketRepository repo, CancellationToken ct)
{
    var ticket = await repo.FindAsync(new TicketId(id), ct);
    return ticket is null ? TypedResults.NotFound() : TypedResults.Ok(TicketSummary.From(ticket));
}
```

---

## 4. Parameter binding

Minimal APIs infer where each parameter comes from:

| Parameter | Bound from |
|---|---|
| Name matches a route parameter (`{id}`) | Route |
| Simple type (int, string, Guid, enum...) otherwise | Query string |
| Registered in DI | Services |
| `HttpContext`, `HttpRequest`, `CancellationToken`, `ClaimsPrincipal` | Special types |
| Complex type (a record/class) | JSON body (for POST/PUT/PATCH) |

Be explicit when inference isn't obvious: `[FromRoute]`, `[FromQuery]`, `[FromHeader]`,
`[FromBody]`, `[FromServices]`, `[FromForm]`, and `[AsParameters]` to bind a whole record
of parameters at once:

```csharp
public sealed record TicketQuery(TicketStatus? Status, string? Assignee, int Page = 1, int PageSize = 20);

app.MapGet("/api/tickets", ([AsParameters] TicketQuery query, ...) => ...);
```

Custom types like `TicketId` can bind from route/query values by implementing
`IParsable<T>` (a static `TryParse`):

```csharp
public readonly record struct TicketId(int Value) : IParsable<TicketId>
{
    public override string ToString() => $"T-{Value}";

    public static bool TryParse(string? s, IFormatProvider? provider, out TicketId result)
    {
        var span = s.AsSpan();
        if (span.StartsWith("T-", StringComparison.OrdinalIgnoreCase)) span = span[2..];
        var ok = int.TryParse(span, out var n) && n > 0;
        result = ok ? new TicketId(n) : default;
        return ok;
    }

    public static TicketId Parse(string s, IFormatProvider? provider)
        => TryParse(s, provider, out var id) ? id : throw new FormatException($"Invalid ticket id '{s}'.");
}
```

Now both `/api/tickets/42` and `/api/tickets/T-42` bind to a `TicketId` parameter, and
anything else returns `400` before your code runs.

---

## 5. Filters

**Endpoint filters** run around a specific endpoint or group, with access to its arguments:
useful for validation, logging or authorization logic that applies to some endpoints only.

```csharp
app.MapPost("/api/tickets", CreateTicket)
   .AddEndpointFilter(async (ctx, next) =>
   {
       var request = ctx.GetArgument<CreateTicketRequest>(0);
       if (request.Title.Length > 200)
           return TypedResults.ValidationProblem(new Dictionary<string, string[]>
               { ["title"] = ["Title must be 200 characters or fewer."] });
       return await next(ctx);
   });
```

Middleware vs filters: **middleware** is for concerns that apply to every request and don't
need endpoint arguments (logging, exception handling, CORS). **Filters** are for
endpoint-specific concerns that need the bound arguments or endpoint metadata.

---

## 6. Error handling in the pipeline

Book I, Chapter 8 established Beacon's policy: expected failures are `Result<T>` values;
unexpected ones are exceptions handled at the boundary. In ASP.NET Core, that boundary is
the exception-handling middleware plus **Problem Details** (RFC 9457), the standard JSON
error format:

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.10",
  "title": "Conflict",
  "status": 409,
  "detail": "Ticket T-42 is closed and can't receive comments.",
  "instance": "/api/tickets/42/comments",
  "traceId": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
}
```

```csharp
builder.Services.AddProblemDetails();     // standard error bodies everywhere
var app = builder.Build();
app.UseExceptionHandler();                // unhandled exceptions → 500 problem details (no stack trace)
app.UseStatusCodePages();                 // bare 404/405 → problem details too
```

For more control, implement `IExceptionHandler` to map specific exception types (for
example, `DomainException` → 409) and to log consistently.

---

## 7. OpenAPI

APIs need machine-readable descriptions for client generation, documentation and testing.
ASP.NET Core generates **OpenAPI** documents from your endpoints:

```csharp
builder.Services.AddOpenApi();
// ...
if (app.Environment.IsDevelopment())
    app.MapOpenApi();                   // serves /openapi/v1.json
```

Pair it with a UI such as Scalar or Swagger UI in development. Typed results, `Produces`
metadata, summaries and descriptions all improve the generated document. Book VII,
Chapter 2 uses this document to generate a typed TypeScript client for the React frontend.

---

## 8. In practice: Beacon.Api

Create the API project and wire it to the existing core:

```bash
dotnet new web -n Beacon.Api -o src/Beacon.Api
dotnet sln add src/Beacon.Api
dotnet add src/Beacon.Api reference src/Beacon.Core
dotnet add src/Beacon.Api package Microsoft.AspNetCore.OpenApi
```

The in-memory repository moves from the CLI into a small infrastructure folder in the API
for now (Book IV replaces it with PostgreSQL):

```csharp
// src/Beacon.Api/Infrastructure/InMemoryTicketRepository.cs
using System.Collections.Concurrent;
using Beacon.Core.Tickets;

namespace Beacon.Api.Infrastructure;

public sealed class InMemoryTicketRepository : ITicketRepository
{
    private readonly ConcurrentDictionary<TicketId, Ticket> _tickets = new();
    private int _lastId;

    public TicketId NextId() => new(Interlocked.Increment(ref _lastId));

    public Task<Ticket?> FindAsync(TicketId id, CancellationToken ct = default)
        => Task.FromResult(_tickets.GetValueOrDefault(id));

    public Task SaveAsync(Ticket ticket, CancellationToken ct = default)
    {
        _tickets[ticket.Id] = ticket;
        return Task.CompletedTask;
    }

    public IReadOnlyList<Ticket> All() => _tickets.Values.ToList();
}
```

Request and response contracts live in the API project. They're the public shape of the
API, separate from domain types, so the domain can change without breaking clients
(Chapter 5 expands on this):

```csharp
// src/Beacon.Api/Tickets/TicketContracts.cs
using Beacon.Core.Tickets;

namespace Beacon.Api.Tickets;

public sealed record CreateTicketRequest(string Title, TicketPriority Priority);
public sealed record AddCommentRequest(string Author, string Body);

public sealed record TicketResponse(
    string Id, string Title, TicketStatus Status, TicketPriority Priority,
    string? Assignee, DateTimeOffset CreatedAt, int CommentCount)
{
    public static TicketResponse From(Ticket t) =>
        new(t.Id.ToString(), t.Title, t.Status, t.Priority, t.Assignee, t.CreatedAt, t.Comments.Count);
}
```

Endpoints are grouped by feature in an extension method:

```csharp
// src/Beacon.Api/Tickets/TicketEndpoints.cs
using Beacon.Api.Infrastructure;
using Beacon.Core.Common;
using Beacon.Core.Tickets;
using Microsoft.AspNetCore.Http.HttpResults;

namespace Beacon.Api.Tickets;

public static class TicketEndpoints
{
    public static IEndpointRouteBuilder MapTicketEndpoints(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/tickets").WithTags("Tickets");

        group.MapGet("/", List);
        group.MapGet("/{id}", Get).WithName("GetTicket");
        group.MapPost("/", Create);
        group.MapPost("/{id}/comments", AddComment);
        return app;
    }

    private static Ok<List<TicketResponse>> List(InMemoryTicketRepository repo)
        => TypedResults.Ok(repo.All().OrderByDescending(t => t.CreatedAt).Select(TicketResponse.From).ToList());

    private static async Task<Results<Ok<TicketResponse>, NotFound>> Get(
        TicketId id, ITicketRepository repo, CancellationToken ct)
    {
        var ticket = await repo.FindAsync(id, ct);
        return ticket is null ? TypedResults.NotFound() : TypedResults.Ok(TicketResponse.From(ticket));
    }

    private static async Task<CreatedAtRoute<TicketResponse>> Create(
        CreateTicketRequest request, InMemoryTicketRepository repo, TimeProvider clock, CancellationToken ct)
    {
        var ticket = new Ticket(repo.NextId(), request.Title, request.Priority, clock.GetUtcNow());
        await repo.SaveAsync(ticket, ct);
        return TypedResults.CreatedAtRoute(TicketResponse.From(ticket), "GetTicket", new { id = ticket.Id.Value });
    }

    private static async Task<IResult> AddComment(
        TicketId id, AddCommentRequest request, TicketService service, CancellationToken ct)
    {
        var result = await service.AddCommentAsync(id, request.Author, request.Body, ct);
        return result.Match(
            t => TypedResults.Ok(TicketResponse.From(t)),
            ToProblem);
    }

    internal static IResult ToProblem(Error error) => error.Code switch
    {
        "not_found"  => TypedResults.Problem(error.Message, statusCode: StatusCodes.Status404NotFound),
        "conflict"   => TypedResults.Problem(error.Message, statusCode: StatusCodes.Status409Conflict),
        "validation" => TypedResults.Problem(error.Message, statusCode: StatusCodes.Status400BadRequest),
        _            => TypedResults.Problem(error.Message, statusCode: StatusCodes.Status500InternalServerError),
    };
}
```

And the composition root:

```csharp
// src/Beacon.Api/Program.cs
using System.Text.Json.Serialization;
using Beacon.Api.Infrastructure;
using Beacon.Api.Tickets;
using Beacon.Core;
using Beacon.Core.Notifications;
using Beacon.Core.Tickets;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddBeaconCore();
builder.Services.AddSingleton<InMemoryTicketRepository>();
builder.Services.AddSingleton<ITicketRepository>(sp => sp.GetRequiredService<InMemoryTicketRepository>());
builder.Services.AddSingleton<INotifier, LoggingNotifier>();

builder.Services.ConfigureHttpJsonOptions(o =>
    o.SerializerOptions.Converters.Add(new JsonStringEnumConverter()));
builder.Services.AddProblemDetails();
builder.Services.AddOpenApi();

var app = builder.Build();

app.UseExceptionHandler();
app.UseStatusCodePages();
if (app.Environment.IsDevelopment())
    app.MapOpenApi();

app.MapTicketEndpoints();
app.MapGet("/health", () => TypedResults.Ok("healthy"));

app.Run();

public partial class Program;   // lets integration tests reference the app (WebApplicationFactory<Program>)
```

```csharp
// src/Beacon.Api/Infrastructure/LoggingNotifier.cs
using Beacon.Core.Notifications;

namespace Beacon.Api.Infrastructure;

public sealed class LoggingNotifier(ILogger<LoggingNotifier> log) : INotifier
{
    public Task NotifyAsync(string recipient, string message, CancellationToken ct = default)
    {
        log.LogInformation("Notify {Recipient}: {Message}", recipient, message);
        return Task.CompletedTask;
    }
}
```

Run it and exercise it with the `.http` file from Chapter 1:

```bash
dotnet run --project src/Beacon.Api
```

```http
POST {{baseUrl}}/api/tickets
Content-Type: application/json

{ "title": "Cannot log in", "priority": "High" }

### → 201 Created, Location: /api/tickets/1

POST {{baseUrl}}/api/tickets/T-1/comments
Content-Type: application/json

{ "author": "maria", "body": "Reset the password." }

### → 200 OK, commentCount: 1

GET {{baseUrl}}/api/tickets/999
### → 404 with a problem details body
```

Notice what we got from the framework and the earlier chapters: `TicketId` binds from
`T-1` or `1`; `TicketService` (scoped) resolves per request; expected failures map to
404/409 via `Result.Match`; unexpected exceptions become a 500 problem-details response
without leaking a stack trace.

### An integration test

```bash
dotnet new xunit -n Beacon.Api.Tests -o tests/Beacon.Api.Tests
dotnet sln add tests/Beacon.Api.Tests
dotnet add tests/Beacon.Api.Tests reference src/Beacon.Api
dotnet add tests/Beacon.Api.Tests package Microsoft.AspNetCore.Mvc.Testing
```

```csharp
// tests/Beacon.Api.Tests/TicketEndpointsTests.cs
using System.Net;
using System.Net.Http.Json;
using Microsoft.AspNetCore.Mvc.Testing;

namespace Beacon.Api.Tests;

public sealed class TicketEndpointsTests(WebApplicationFactory<Program> factory)
    : IClassFixture<WebApplicationFactory<Program>>
{
    [Fact]
    public async Task Create_then_get_returns_the_ticket()
    {
        var client = factory.CreateClient();

        var create = await client.PostAsJsonAsync("/api/tickets", new { title = "VPN down", priority = "Urgent" });
        Assert.Equal(HttpStatusCode.Created, create.StatusCode);

        var get = await client.GetAsync(create.Headers.Location);
        Assert.Equal(HttpStatusCode.OK, get.StatusCode);
        var body = await get.Content.ReadFromJsonAsync<Dictionary<string, object>>();
        Assert.Equal("VPN down", body!["title"]?.ToString());
    }

    [Fact]
    public async Task Unknown_ticket_returns_problem_details_404()
    {
        var response = await factory.CreateClient().GetAsync("/api/tickets/T-99999");
        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
        Assert.Equal("application/problem+json", response.Content.Headers.ContentType?.MediaType);
    }
}
```

`WebApplicationFactory` runs the real pipeline in memory (routing, binding, serialization,
error handling), the seams Book I, Chapter 14 said deserve integration tests.

---

## 9. What can go wrong

- **Middleware in the wrong order**: authorization before authentication; exception handler
  added late so it doesn't catch earlier failures.
- **Modifying headers after the response started.**
- **Blocking calls** in handlers (`.Result`), causing thread-pool starvation (Book I,
  Chapter 9).
- **Ambiguous routes** (`/api/tickets/{id}` and `/api/tickets/{slug}`) throwing at runtime.
- **Leaking exception details** in production (developer exception page enabled outside
  Development).
- **Returning domain entities directly** from endpoints, exposing internals and coupling
  clients to your domain.
- **Fat endpoints** with business logic in lambdas instead of services.
- **Ignoring `CancellationToken`**, so abandoned requests keep consuming resources.

---

## 10. How an experienced engineer thinks about this

- **Know the pipeline.** When behavior is surprising, ask: which middleware ran, in what
  order, and which endpoint matched?
- **Keep endpoints thin.** Bind, call a service, map the result. Business logic lives in
  the core.
- **Contracts are not domain models.** Request/response types are a public API with their
  own lifecycle.
- **Standardize errors** with Problem Details, so clients handle failures uniformly.
- **Test through the real pipeline** with `WebApplicationFactory` for routing, binding and
  serialization.

---

## 11. Check yourself

**Questions**

1. What does Kestrel do, and why is it often behind a reverse proxy?
2. What is middleware? Why does the order of `UseX` calls matter?
3. How does a minimal API decide where to bind each parameter from?
4. When would you use an endpoint filter instead of middleware?
5. What are Problem Details, and why standardize on them?
6. Why separate request/response contracts from domain types?

**Exercises**

1. Add `POST /api/tickets/{id}/assign` and `POST /api/tickets/{id}/resolve` endpoints using
   new `TicketService` methods that return `Result<Ticket>`.
2. Write a middleware that adds an `X-Request-Id` header (generate one if the request
   doesn't have it) using `OnStarting`.
3. Implement an `IExceptionHandler` that maps `DomainException` to `409` problem details,
   and test it with `WebApplicationFactory`.
4. Add Scalar (or Swagger UI) in Development and explore the generated OpenAPI document.

**Interview-style questions**

- "Walk me through what happens when an HTTP request reaches an ASP.NET Core app."
- "What's the difference between middleware and filters?"
- "Minimal APIs or controllers? Why?"

---

## 12. Going deeper

- [Microsoft docs: ASP.NET Core fundamentals](https://learn.microsoft.com/aspnet/core/fundamentals/)
- [Microsoft docs: Minimal APIs overview](https://learn.microsoft.com/aspnet/core/fundamentals/minimal-apis/overview)
- [Microsoft docs: Middleware](https://learn.microsoft.com/aspnet/core/fundamentals/middleware/)
- Andrew Lock, *ASP.NET Core in Action* and his blog (andrewlock.net).

**Next:** [Chapter 3 — Configuration, Options and Logging](03-configuration-options-and-logging.md)
covers how Beacon's API is configured per environment and how it reports what it's doing.
