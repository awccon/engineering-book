# Background Work and Real-Time

Not all work belongs in the request/response cycle. Sending an email shouldn't make the
user wait for an SMTP server. Recalculating SLA breaches should happen every minute
whether or not anyone is making requests. And when a ticket is updated, the agent looking
at it shouldn't have to press refresh to find out.

This chapter covers two directions of the same idea: moving work **out of** the request
(background processing), and pushing updates **to** clients without a request
(real-time communication).

---

## 1. The problem: the request is the wrong place for some work

A request handler should do the minimum needed to give the client a correct answer, then
return. Work that doesn't fit:

| Kind of work | Example | Why not in the request? |
|---|---|---|
| Slow side effects | Sending emails, calling webhooks | User waits for something they don't need to wait for; failures fail the request |
| Scheduled work | SLA breach checks every minute | No request triggers it |
| Long-running jobs | Exporting 100,000 tickets, reindexing search | Exceeds HTTP timeouts |
| Retryable work | Calling a flaky partner API | Retrying inside a request makes it slower and still loses work on crash |

And the inverse: clients need to learn about changes they didn't request ("a new comment
arrived"). Polling works, but wastes resources and adds delay.

---

## 2. The mental model

### Three places background work can run

```text
 1. In-process, fire-and-forget     ✗  Task.Run(...) from a request: lost on crash, no retries, unobserved errors
 2. In-process hosted service       ✓  BackgroundService + in-memory queue: simple, but lost on crash/redeploy
 3. Durable queue + worker          ✓✓ Message broker or DB-backed jobs: survives crashes, retries, scales out
```

The right choice depends on one question: **what happens if this work is lost?**

- A cache warm-up that gets lost: harmless. In-process is fine.
- A "your ticket was resolved" email that gets lost: annoying. In-process may be acceptable;
  durable is better.
- A payment capture or an invoice that gets lost: unacceptable. Must be durable.

> **🧱 Durable:** Anything that must happen *eventually, exactly as requested* needs to be
> **written down durably before the request returns**: in a database table or a message
> broker. In-memory queues disappear on every deployment.

---

## 3. Hosted services in ASP.NET Core

An `IHostedService` runs alongside the web server, started and stopped by the host.
`BackgroundService` is the convenient base class:

```csharp
public sealed class SlaMonitor(
    IServiceScopeFactory scopes,
    IOptions<SlaOptions> options,
    ILogger<SlaMonitor> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var timer = new PeriodicTimer(options.Value.BreachCheckInterval);
        while (await timer.WaitForNextTickAsync(stoppingToken))
        {
            try
            {
                await using var scope = scopes.CreateAsyncScope();
                var checker = scope.ServiceProvider.GetRequiredService<SlaBreachChecker>();
                await checker.RunAsync(stoppingToken);
            }
            catch (OperationCanceledException) when (stoppingToken.IsCancellationRequested)
            {
                break;
            }
            catch (Exception ex)
            {
                log.LogError(ex, "SLA check failed; will retry next tick");   // one failure mustn't kill the loop
            }
        }
    }
}

builder.Services.AddHostedService<SlaMonitor>();
```

Key points, each preventing a real bug:

- **Hosted services are singletons**, so scoped dependencies (a `DbContext`) need a scope
  per iteration (Book I, Chapter 13).
- **Catch exceptions inside the loop.** An unhandled exception in `ExecuteAsync` stops the
  service (and, since .NET 6, by default stops the whole host).
- **Honor `stoppingToken`** for graceful shutdown. The host waits a limited time (30 seconds
  by default, `HostOptions.ShutdownTimeout`) for services to stop.
- **`PeriodicTimer`** is async-friendly and doesn't overlap runs: if a run takes longer than
  the interval, the next tick waits.

### Multiple instances

If Beacon.Api runs on three instances, `SlaMonitor` runs **three times** every minute.
Options:

- Make the work **idempotent** so duplicates are harmless (marking a ticket as breaching
  twice is fine).
- Use a **distributed lock** or **leader election** (a database advisory lock, a Redis
  lock, a lease in blob storage).
- Run scheduled work in a **separate worker process** with one instance.
- Use a scheduler that handles this (Hangfire, Quartz.NET with clustering, or a cloud
  scheduler triggering an endpoint or function).

---

## 4. Queues

### In-process: Channels

For work that can be lost but should be fast and off the request path, a hosted service
consuming a `Channel<T>` is ideal. Beacon already has `IndexingQueue` (Book I, Chapter 10):

```csharp
public sealed class IndexingWorker(IndexingQueue queue, IServiceScopeFactory scopes, ILogger<IndexingWorker> log)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var id in queue.ReadAllAsync(stoppingToken))
        {
            try
            {
                await using var scope = scopes.CreateAsyncScope();
                await scope.ServiceProvider.GetRequiredService<ITicketIndexer>().IndexAsync(id, stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                log.LogError(ex, "Failed to index {TicketId}", id);
            }
        }
    }
}
```

### Durable: message brokers and database-backed jobs

When work must not be lost:

| Option | Examples | Notes |
|---|---|---|
| Message broker | Azure Service Bus, RabbitMQ, Amazon SQS | Durable queues, retries, dead-lettering, scale-out consumers (Book XIII, Chapter 4) |
| Job library on your DB | Hangfire, Quartz.NET (persistent store) | Jobs stored in your database; dashboards, retries, scheduling |
| Database table as queue | `jobs` table + `SELECT ... FOR UPDATE SKIP LOCKED` (PostgreSQL) | Minimal infrastructure; works well at moderate volume (Book IV, Chapter 5) |

### Delivery guarantees

Distributed queues offer **at-least-once** delivery: a message may be delivered more than
once (the consumer crashed after doing the work but before acknowledging it). Therefore:

> **⚠️ What can go wrong:** Consumers must be **idempotent**. Processing "send resolution
> email for T-42" twice sends two emails unless the handler records that it already did it.
> Track processed message IDs, or make the operation naturally idempotent ("set status to
> X" rather than "increment counter").

### The dual-write problem

A subtle and very common bug:

```csharp
await db.SaveChangesAsync();              // 1. ticket resolved in the database
await bus.PublishAsync(new TicketResolved(...));   // 2. message published
```

If the process crashes between 1 and 2, the ticket is resolved but no one is notified.
Reversing the order has the opposite problem. You can't atomically write to a database
and a broker.

The standard solution is the **transactional outbox**: write the message to an `outbox`
table **in the same database transaction** as the business change; a background process
reads the outbox and publishes. Book IV, Chapter 5 implements it, and Book XIII, Chapter 4
discusses it in depth.

---

## 5. Long-running operations over HTTP

When a client requests something that takes minutes (an export), use the
**asynchronous request-reply** pattern:

```text
POST /api/exports            → 202 Accepted, Location: /api/exports/9f2c
GET  /api/exports/9f2c       → 200 { "status": "running", "progress": 40 }
GET  /api/exports/9f2c       → 200 { "status": "completed", "downloadUrl": "..." }
```

The job is queued durably and processed by a worker; the client polls (or receives a
real-time notification).

---

## 6. Real-time: pushing to clients

### Options

| Technique | Direction | Transport | Good for |
|---|---|---|---|
| Polling | Client → server, repeatedly | HTTP | Low-frequency updates; simplest |
| Long polling | Server holds request until news | HTTP | Legacy fallback |
| **Server-Sent Events (SSE)** | Server → client | One long HTTP response, `text/event-stream` | Notifications, feeds, streaming AI output |
| **WebSockets** | Both directions | Upgraded TCP connection | Chat, collaboration, games |
| **SignalR** | Both directions | Abstraction over WebSockets/SSE/long polling | .NET real-time apps with groups, reconnection, scale-out |

### Server-Sent Events

SSE is plain HTTP: the server keeps the response open and writes events:

```text
event: ticketUpdated
data: {"id":"T-42","status":"Resolved"}

event: commentAdded
data: {"ticketId":"T-42","author":"maria"}
```

Browsers support it natively with `EventSource`, including automatic reconnection.

> **🔄 Current (as of October 2026):** ASP.NET Core 10 adds `TypedResults.ServerSentEvents`,
> which streams an `IAsyncEnumerable<T>` (or `SseItem<T>` values with event names) as SSE.

### SignalR

SignalR provides **hubs**: server classes clients connect to, with methods callable in both
directions, **groups** (send to everyone watching ticket T-42), automatic transport
negotiation and reconnection, and typed clients.

```csharp
public interface ITicketClient
{
    Task TicketUpdated(TicketResponse ticket);
    Task CommentAdded(string ticketId, string author);
}

[Authorize]
public sealed class TicketHub : Hub<ITicketClient>
{
    public Task Watch(string ticketId) => Groups.AddToGroupAsync(Context.ConnectionId, $"ticket:{ticketId}");
    public Task Unwatch(string ticketId) => Groups.RemoveFromGroupAsync(Context.ConnectionId, $"ticket:{ticketId}");
}
```

Sending from anywhere in the app through `IHubContext<TicketHub, ITicketClient>`:

```csharp
await hub.Clients.Group($"ticket:{ticket.Id}").TicketUpdated(TicketResponse.From(ticket));
```

> **⚠️ What can go wrong:** `Watch(ticketId)` above lets any authenticated user subscribe
> to *any* ticket's updates: the same IDOR problem as Chapter 6, over a different
> transport. Authorize group membership against the resource, just like an HTTP endpoint.

### Scaling real-time

Each instance only knows about its own connections. With several instances behind a load
balancer, a message sent on instance A won't reach a client connected to instance B. Solve
it with a **backplane** (Redis for SignalR) or a managed service (Azure SignalR Service),
which also offloads connection handling. Long-lived connections also need load balancer
support (WebSockets, timeouts) and sticky sessions in some configurations.

> **🧭 When not to use real-time:** If data changes every few minutes and users don't
> need instant updates, polling every 30–60 seconds is simpler to build, scale and debug.
> Real-time is worth it when immediacy is part of the product: chats, live dashboards,
> collaborative editing, an agent seeing a customer's reply as it arrives.

---

## 7. In practice: Beacon's background and real-time features

### Notifications off the request path

Assigning a ticket currently calls `INotifier` synchronously inside the request. We'll
move notification to a background worker fed by domain events (Book I, Chapter 6).

```csharp
// src/Beacon.Api/Notifications/NotificationQueue.cs
using System.Threading.Channels;

namespace Beacon.Api.Notifications;

public sealed record NotificationJob(string Recipient, string Message);

public sealed class NotificationQueue
{
    private readonly Channel<NotificationJob> _channel =
        Channel.CreateBounded<NotificationJob>(new BoundedChannelOptions(10_000) { FullMode = BoundedChannelFullMode.Wait });

    public ValueTask EnqueueAsync(NotificationJob job, CancellationToken ct) => _channel.Writer.WriteAsync(job, ct);
    public IAsyncEnumerable<NotificationJob> ReadAllAsync(CancellationToken ct) => _channel.Reader.ReadAllAsync(ct);
}
```

```csharp
// src/Beacon.Api/Notifications/NotificationWorker.cs
namespace Beacon.Api.Notifications;

public sealed class NotificationWorker(NotificationQueue queue, INotifier notifier, ILogger<NotificationWorker> log)
    : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await foreach (var job in queue.ReadAllAsync(stoppingToken))
        {
            try
            {
                await notifier.NotifyAsync(job.Recipient, job.Message, stoppingToken);
            }
            catch (Exception ex) when (ex is not OperationCanceledException)
            {
                log.LogWarning(ex, "Notification to {Recipient} failed", job.Recipient);
            }
        }
    }
}
```

Domain-event handlers registered with the `EventDispatcher` now *enqueue* instead of
sending:

```csharp
dispatcher.On<TicketAssigned>((e, ct) =>
    queue.EnqueueAsync(new NotificationJob(e.Assignee, $"You were assigned {e.TicketId}."), ct).AsTask());
```

The request returns as soon as the job is queued. This is an **explicitly lossy** design:
a crash loses queued notifications. That's acceptable for Beacon's in-app notifications
for now, and the code makes the decision visible. Book IV upgrades it to a transactional
outbox for emails that must be sent.

### Live ticket updates with SignalR

```csharp
// Program.cs (excerpt)
builder.Services.AddSignalR();
// ...
app.MapHub<TicketHub>("/hubs/tickets");
```

The hub authorizes watching:

```csharp
// src/Beacon.Api/Realtime/TicketHub.cs
using Beacon.Api.Security;
using Beacon.Api.Tickets;
using Beacon.Core.Tickets;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.SignalR;

namespace Beacon.Api.Realtime;

public interface ITicketClient
{
    Task TicketUpdated(TicketResponse ticket);
}

[Authorize]
public sealed class TicketHub(ITicketRepository repo, IAuthorizationService authz) : Hub<ITicketClient>
{
    public async Task Watch(string ticketId)
    {
        if (!TicketId.TryParse(ticketId, null, out var id)) throw new HubException("Invalid ticket id.");

        var ticket = await repo.FindAsync(id);
        if (ticket is null || !(await authz.AuthorizeAsync(Context.User!, ticket, TicketOperations.Read)).Succeeded)
            throw new HubException("Ticket not found.");      // same answer as the HTTP API

        await Groups.AddToGroupAsync(Context.ConnectionId, Group(id));
    }

    public static string Group(TicketId id) => $"ticket:{id}";
}
```

And a domain-event handler broadcasts changes:

```csharp
dispatcher.On<TicketResolved>(async (e, ct) =>
{
    var ticket = await repo.FindAsync(e.TicketId, ct);
    if (ticket is not null)
        await hub.Clients.Group(TicketHub.Group(e.TicketId)).TicketUpdated(TicketResponse.From(ticket));
});
```

Book VI connects the React frontend to this hub, so an agent's ticket view updates live.

### SLA monitoring

`SlaMonitor` from section 3 runs on a timer and raises a `TicketBreachedSla` domain event
for newly breaching tickets (tracking which tickets were already flagged, so repeated runs
and multiple instances don't flag twice: idempotency again).

---

## 8. What can go wrong

- **Fire-and-forget `Task.Run`** from requests: lost work, unobserved exceptions, work cut
  off at shutdown.
- **Unhandled exceptions killing background loops** (or the host).
- **Scoped services captured by singleton workers.**
- **Duplicate scheduled work** across multiple instances.
- **Non-idempotent consumers** with at-least-once delivery.
- **Dual writes** to database and broker.
- **Unbounded in-memory queues** under load.
- **Unauthorized real-time subscriptions.**
- **Real-time that doesn't scale out** without a backplane.
- **Connection limits and proxy timeouts** killing long-lived connections.

---

## 9. How an experienced engineer thinks about this

- **Ask what happens if the work is lost**, then choose in-memory or durable.
- **Assume duplicates.** Make background work idempotent.
- **Assume multiple instances.** Scheduled jobs and in-memory state must account for it.
- **Real-time channels are API surface** with the same authentication and authorization
  needs as HTTP endpoints.
- **Prefer simple**: polling before WebSockets, a DB-backed job before a broker, until the
  requirements demand more.

---

## 10. Check yourself

**Questions**

1. Why is `Task.Run` from a request handler a bad way to do background work?
2. How should a `BackgroundService` use a `DbContext`?
3. What happens to a scheduled background service when you run three instances?
4. What is at-least-once delivery, and what does it require of consumers?
5. What's the dual-write problem? How does an outbox solve it?
6. Compare SSE, WebSockets and SignalR.
7. Why does SignalR need a backplane with multiple instances?

**Exercises**

1. Implement `SlaMonitor` with a fake clock in tests (`FakeTimeProvider` can drive
   `PeriodicTimer` when the timer is created from the provider).
2. Add an SSE endpoint `GET /api/tickets/{id}/events` using `TypedResults.ServerSentEvents`
   as an alternative to SignalR.
3. Write an integration test that connects a SignalR client (`Microsoft.AspNetCore.SignalR.Client`)
   through `WebApplicationFactory`, watches a ticket, resolves it, and receives the update.
4. Implement an async export: `POST /api/exports` → 202, a worker that writes a CSV, and a
   status endpoint.

**Interview-style questions**

- "How do you run background jobs in ASP.NET Core?"
- "How do you guarantee that a message is published when a database transaction commits?"
- "How would you build real-time notifications for a web app that runs on several servers?"

---

## 11. Going deeper

- [Microsoft docs: Background tasks with hosted services](https://learn.microsoft.com/aspnet/core/fundamentals/host/hosted-services)
- [Microsoft docs: ASP.NET Core SignalR](https://learn.microsoft.com/aspnet/core/signalr/introduction)
- [Azure Architecture Center: Asynchronous Request-Reply pattern](https://learn.microsoft.com/azure/architecture/patterns/async-request-reply)
- [microservices.io: Transactional outbox](https://microservices.io/patterns/data/transactional-outbox.html)

**Next:** [Chapter 9 — Securing a Web API](09-securing-a-web-api.md) reviews Beacon's API
against the threats it will face.
