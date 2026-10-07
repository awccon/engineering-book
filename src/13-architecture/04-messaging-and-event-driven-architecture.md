# Messaging and Event-Driven Architecture

Chapter 3 ended with a recommendation: move work that doesn't need to happen *now* off the request path
and behind a queue. Beacon already does this in a small way. Domain events go into the outbox table in
the same transaction as the ticket change (Book IV, Chapter 5), and a worker processes them. This chapter
generalizes that into **messaging**: queues, topics and logs, the guarantees they give and don't give,
how events connect modules and services, how multi-step processes work without distributed
transactions, and what eventual consistency means for users.

---

## 1. The problem: synchronous coupling

When ticket resolution has to update the search index, notify the customer, send a webhook, update the
analytics dashboard and trigger a survey, the naïve design calls each of them in the request:

```text
Resolve ticket ─► update DB ─► reindex ─► email ─► webhook ─► analytics ─► survey ─► 200 OK
```

Every arrow is a problem from Chapter 3:

- **Availability multiplies**: if any of five dependencies is down, resolving fails (or half-succeeds).
- **Latency adds**: the agent waits for the slowest dependency.
- **Coupling grows**: the Tickets module knows about every consumer, and adding one means changing it.
- **Load is coupled**: a burst of resolutions becomes a burst on every downstream system at once.

Messaging breaks these chains. The Tickets module records **what happened** and moves on; each
interested party reacts in its own time, at its own pace, with its own retries.

---

## 2. The mental model: commands, events and the broker in between

Three kinds of message, with different meanings and different coupling:

| Kind | Meaning | Naming | Receivers | Example |
|---|---|---|---|---|
| **Command** | "Please do this" | Imperative | Exactly one handler, which may refuse | `SendSurvey`, `ReindexArticle` |
| **Event** | "This happened" | Past tense | Zero or more subscribers; can't be refused | `TicketResolved`, `ArticlePublished` |
| **Query** | "Tell me this" | Question | One responder | Usually HTTP, rarely messaging |

The distinction matters for ownership. A **command** is owned by the receiver (the receiver defines what
it accepts); the sender depends on the receiver. An **event** is owned by the publisher (it defines what
it announces); subscribers depend on the publisher, and the publisher doesn't know they exist.

Between sender and receiver sits a **broker** (or a table acting as one), which provides:

- **Temporal decoupling**: the receiver doesn't have to be running when the message is sent.
- **Load leveling**: bursts are absorbed into a backlog and processed at a sustainable rate.
- **Retries and dead-lettering**: failed processing is retried, and messages that keep failing are set
  aside.
- **Fan-out**: one event, many independent subscribers.

> **🧱 Durable:** A broker trades **immediacy** for **independence**. The sender no longer knows when, or
> whether yet, the work happened. Everything in this chapter is about getting back the guarantees you
> need (delivery, ordering, consistency, visibility) without giving up that independence.

---

## 3. Queues, topics and logs

Three broker models, each suited to different jobs:

### Queues (point-to-point)

Messages go into a queue; **competing consumers** take them off, each message processed by one consumer.
Adding consumers scales throughput. Typical for commands and work distribution.

```text
Producer ─► [ m5 m4 m3 m2 m1 ] ─┬─► Consumer A  (gets m1, m3)
                                └─► Consumer B  (gets m2, m4)
```

### Topics (publish/subscribe)

A message published to a **topic** is copied to every **subscription**; each subscription behaves like
its own queue with its own consumers. Typical for events.

```text
                       ┌─► subscription "notifications" ─► Notifications workers
TicketResolved ─► topic ┼─► subscription "search"        ─► Indexing workers
                       └─► subscription "webhooks"      ─► beacon-relay
```

### Logs (streams)

A **log** (Kafka, Azure Event Hubs, Redpanda) is an append-only, partitioned, retained sequence of
records. Consumers don't remove messages; each **consumer group** tracks its own position (offset) and can
re-read history. Order is guaranteed within a partition.

```text
partition 0: [ e0 e1 e2 e3 e4 e5 e6 ... ]   ← group "analytics" at offset 3
                                            ← group "search"    at offset 6
```

| | Queue / topic (Service Bus, RabbitMQ, SQS/SNS) | Log (Kafka, Event Hubs) |
|---|---|---|
| Message lifetime | Until processed (or TTL) | Retained by time/size, re-readable |
| Per-message features | Ack per message, delayed delivery, dead-letter, sessions | Commit offsets; per-message retry is your job |
| Ordering | Per session/partition key (if enabled) | Per partition |
| Throughput | Thousands–tens of thousands/s | Millions/s |
| Best for | Business workflows, commands, integration events | Telemetry, event streams, replay, analytics pipelines |

For an application like Beacon, a **queue/topic broker** fits business messaging; a **log** fits
high-volume telemetry and analytics. Many organizations run both.

> **🔄 Current (as of October 2026):** On Azure, **Service Bus** is the business message broker (queues,
> topics, sessions, dead-letter queues, scheduled messages, duplicate detection), **Event Hubs** is the
> managed log with a Kafka-compatible endpoint, and **Event Grid** routes lightweight notifications
> (including Azure resource events) to handlers. RabbitMQ remains the most common self-hosted broker;
> Kafka dominates streaming. AWS's equivalents are SQS/SNS, Kinesis/MSK and EventBridge.

### PostgreSQL as a queue

For moderate volumes, **a table is a perfectly good queue**. Beacon's outbox already uses `FOR UPDATE SKIP
LOCKED` (Book IV, Chapter 5) to let several workers claim rows without blocking each other. Advantages:
transactional with your data, no new infrastructure, easy to inspect with SQL. Limits: polling latency
(or `LISTEN/NOTIFY` to reduce it), table bloat at high churn (vacuum tuning, partitioning), and no fan-out
or cross-service delivery without extra work. Use a broker when you need fan-out to many consumers,
cross-service delivery, or volumes where polling hurts.

---

## 4. Delivery guarantees in practice

Recall from Chapter 3: over an unreliable network you get **at most once** or **at least once**, and
"exactly once" is at-least-once plus idempotent handling.

Brokers implement at-least-once with **acknowledgments**:

1. The consumer receives a message; the broker **locks** it (Service Bus "peek-lock", SQS "visibility
   timeout").
2. The consumer processes it.
3. The consumer **completes** (acks) it; the broker deletes it.
4. If the consumer crashes or the lock expires first, the message becomes visible again and is
   **redelivered**.

So redelivery happens whenever processing succeeds but the ack is lost, the consumer crashes after side
effects, or processing takes longer than the lock. **Every handler must be idempotent.**

### Poison messages and dead-letter queues

A message that fails **every** time (a bug, malformed data, a reference to a deleted ticket) would be
retried forever, blocking or wasting capacity. Brokers count deliveries and, after a limit (Service Bus
`MaxDeliveryCount`, default 10), move the message to a **dead-letter queue (DLQ)**.

A DLQ is only useful if someone looks at it:

- **Alert** on DLQ depth > 0 for business-critical queues.
- **Record why**: dead-letter with a reason and description (exception type, message).
- **Have a replay tool**: after fixing the bug, move messages back to the main queue. `beacon-tools`
  (Book X) is a natural home for a `beacon dlq replay --subscription notifications --dry-run` command.
- **Distinguish transient and permanent failures** in the handler: retry (abandon) on a timeout,
  dead-letter immediately on a validation error that will never succeed.

### Ordering

Global ordering doesn't scale: it means one consumer at a time. Brokers instead order **within a key**:
Service Bus **sessions**, Kafka **partitions**, SQS FIFO **message groups**. Choose the key so that
messages that must be ordered share it, typically the entity ID (`TicketId`). Different tickets are
processed in parallel; events for one ticket arrive in order.

Even with ordered delivery, a retry or a redelivery can reorder effects. Robust consumers also carry a
**version** (Beacon's `Ticket.Version`) and ignore events older than what they've already applied.

---

## 5. The outbox and the inbox

### Why you need an outbox

The **dual-write problem**: updating the database and publishing a message are two separate systems. If
you do both, one can succeed and the other fail:

```csharp
await db.SaveChangesAsync(ct);                 // ✓ committed
await sender.SendMessageAsync(evt, ct);        // ✗ broker unreachable → event lost forever
```

Reversing the order is worse: the event is published, the commit fails, and subscribers react to
something that never happened.

The **transactional outbox** writes the message into an `outbox` table **in the same transaction** as the
business change. A separate **relay** reads the outbox and publishes to the broker, marking rows as sent.
If publishing fails, it retries; the event is never lost, but may be published **more than once** (crash
after publish, before marking). Beacon has done this since Book IV via `DomainEventsToOutboxInterceptor`.

```text
┌───────────── one PostgreSQL transaction ─────────────┐
│ UPDATE tickets SET status = 'resolved', version = 13 │
│ INSERT INTO outbox (id, type, payload, ...)          │
└──────────────────────────────────────────────────────┘
                  │
     Outbox relay (claims rows with SKIP LOCKED, publishes, marks sent)
                  ▼
      Service Bus topic "beacon-events"
```

An alternative relay is **change data capture (CDC)**: tools like Debezium read PostgreSQL's
write-ahead log (logical replication) and publish outbox rows without polling. More infrastructure,
lower latency, no polling load.

### The inbox (idempotent consumer)

On the receiving side, an **inbox** table records processed message IDs **in the same transaction** as
the handler's effects:

```sql
CREATE TABLE notifications.inbox (
    message_id   uuid        NOT NULL,
    handler      text        NOT NULL,      -- one message can feed several handlers
    processed_at timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (message_id, handler)
);
```

```csharp
public async Task HandleAsync(InboundMessage msg, CancellationToken ct)
{
    await using var tx = await db.Database.BeginTransactionAsync(ct);

    var first = await db.Database.ExecuteSqlAsync($"""
        INSERT INTO notifications.inbox (message_id, handler) VALUES ({msg.Id}, {HandlerName})
        ON CONFLICT DO NOTHING
        """, ct) == 1;
    if (!first) return;                         // duplicate: already applied (tx rolls back, harmless)

    await ApplyAsync(msg, ct);                  // database effects of the handler
    await db.SaveChangesAsync(ct);
    await tx.CommitAsync(ct);
}
```

This makes database effects exactly-once. **External** effects (sending an email, calling a webhook)
can't join the transaction; for those, write an outbox row from the handler (so the send itself is
reliable) and pass an idempotency key to the external system (Chapter 3).

---

## 6. Designing events

### Notification vs state transfer

Two styles of event payload:

- **Event notification**: minimal data. `{ "ticketId": "T-42", "version": 13 }`. Subscribers call back to
  get details. Small, never stale, but creates load and runtime coupling on the publisher.
- **Event-carried state transfer**: the event includes what subscribers need.
  `{ "ticketId": "T-42", "status": "resolved", "resolution": "fixed", "reporterId": "...", "teamId": "..." }`.
  Subscribers can work without calling back (and keep local copies), at the cost of larger events and a
  bigger contract.

Beacon's integration events carry the state most subscribers need, which lets Notifications, Search and
`beacon-relay` work without calling the Tickets module.

### Events are contracts

Once published, an event's schema is a public API (Book VII, Chapter 2). Rules that keep it evolvable:

- **Add optional fields; never remove or rename** without a versioned replacement.
- Consumers must **ignore unknown fields** (tolerant reader).
- Include **metadata**: event ID, type, version, occurred-at, source, correlation and trace context.
- Version by type name when breaking: `beacon.ticket.resolved.v2`, published alongside v1 until all
  consumers migrate.
- Consider a **schema registry** or published JSON Schemas generated from code when many teams consume.

The **CloudEvents** specification (CNCF) standardizes the envelope (`id`, `source`, `type`, `time`,
`datacontenttype`, `data`), and Event Grid supports it natively. Beacon uses CloudEvents-style attribute
names for its integration events, which also makes `beacon-relay`'s webhooks recognizable to customers.

### Internal domain events vs integration events

Keep two kinds separate (Chapter 2):

- **Domain events** (`TicketResolved` record in `Beacon.Core`) are internal, rich types, free to change
  with the code.
- **Integration events** (`TicketResolvedIntegrationEvent` in `Beacon.Tickets.Contracts`) are published
  contracts, mapped from domain events at the boundary, versioned carefully.

Publishing domain events directly couples every subscriber to your internal model.

### Event sourcing, briefly

**Event sourcing** goes further: the events *are* the source of truth. A ticket isn't a row with a
status; it's the sequence `TicketCreated, TicketAssigned, CommentAdded, TicketResolved`, and current
state is computed by replaying them (with snapshots for speed). Benefits: a perfect audit trail, the
ability to rebuild read models and answer new questions about the past. Costs: event schema evolution
forever, more complex queries (you need projections), harder GDPR deletion, and an unfamiliar model for
most developers.

> **🧭 When not to use it:** Event sourcing is valuable for domains where history *is* the business
> (ledgers, compliance-heavy workflows) and a poor default elsewhere. Beacon keeps state in tables and
> gets most of the benefit with an audit log (Book I) and published events. Don't confuse
> **event-driven** (communicating with events) with **event-sourced** (storing state as events); you can
> have the first without the second.

---

## 7. Sagas: multi-step processes without distributed transactions

Some business processes span several modules or services, each with its own data: "When an
organization upgrades to Business, provision SSO, enable the AI assistant, migrate their SLA targets and
send a welcome email." There's no transaction across all of these. If step 3 fails, steps 1 and 2 have
already happened.

A **saga** is a sequence of local transactions, each publishing an event or command that triggers the
next, with **compensating actions** to undo completed steps if a later step fails. Compensation is
semantic, not a rollback: you can't un-send an email, but you can send a correction; you can't un-charge
a card, but you can refund it.

Two coordination styles:

**Choreography**: each participant reacts to events and emits its own. No central coordinator.

```text
Billing: PlanUpgraded ─► Identity: provisions SSO ─► SsoProvisioned ─► Assistant: enables AI ─► ...
```

Simple for two or three steps; for longer flows, the process exists only implicitly across many
handlers, and "where is this upgrade stuck?" becomes hard to answer.

**Orchestration**: a coordinator (a **process manager**) sends commands, waits for replies, tracks state
and decides what's next, including compensation.

```csharp
// A persisted state machine; each transition is one local transaction + outbox messages
public sealed class PlanUpgradeProcess
{
    public Guid Id { get; init; }
    public OrganizationId Org { get; init; }
    public UpgradeStep Step { get; private set; } = UpgradeStep.Started;
    public DateTimeOffset Deadline { get; init; }

    public IEnumerable<object> On(object message) => (Step, message) switch
    {
        (UpgradeStep.Started, PlanUpgraded)        => Advance(UpgradeStep.ProvisioningSso, new ProvisionSso(Org)),
        (UpgradeStep.ProvisioningSso, SsoProvisioned) => Advance(UpgradeStep.EnablingAssistant, new EnableAssistant(Org)),
        (UpgradeStep.EnablingAssistant, AssistantEnabled) => Advance(UpgradeStep.Done, new SendWelcomeEmail(Org)),
        (_, StepFailed f)                          => Compensate(f),
        (_, DeadlinePassed)                        => Compensate(new StepFailed("timeout")),
        _                                          => [],          // duplicate or stale message: ignore
    };
    // Advance/Compensate update Step and return the commands to send via the outbox
}
```

Orchestration makes the process explicit, observable and testable, at the cost of a central component.
Prefer it for flows with more than a few steps, timeouts or compensation. Durable workflow engines
(Temporal, Azure Durable Functions / Durable Task, Dapr Workflow) provide orchestration with persisted
state and timers so you don't build the machinery yourself.

> **⚠️ What can go wrong:** Sagas expose **intermediate states**: for a while, the organization has SSO
> but no AI assistant. Design the UI and other processes to tolerate those states ("Upgrade in
> progress"), and make every step and compensation idempotent, because messages will be redelivered.

---

## 8. Eventual consistency and the user

With messaging, parts of the system lag behind others. The ticket is resolved immediately; the search
index, dashboard and customer's email follow within seconds (or, during an incident, minutes). That's
**eventual consistency**, and it's mostly a **user experience** problem:

- **Return the result of the write directly** so the user who acted sees their change immediately
  (read-your-writes), rather than re-reading a lagging projection.
- **Show pending states honestly**: "Survey will be sent", "Indexing…", "Suggested priority: analyzing".
- **Push updates** when the background work completes. Beacon's SignalR hub (Book III) already does this.
- **Show freshness** for derived views: "Dashboard updated 40 s ago".
- **Monitor lag as a metric** (outbox age, subscription backlog, consumer lag) with alerts. Eventual
  consistency without a bound is just inconsistency.

> **🧱 Durable:** Ask the business, not the database: "How stale can this be before it causes harm?"
> Most answers are seconds or minutes, which messaging delivers easily. The few answers of "never"
> (account balance, stock reservation, permission revocation) belong in a single transactional store.

---

## 9. In practice: Beacon's event backbone

As Beacon became a modular monolith with satellites (Chapter 2), the outbox relay started publishing
integration events to an Azure Service Bus topic so that out-of-process consumers (`beacon-relay`,
analytics, `beacon-tools`) and in-process modules use the same mechanism.

```text
Modules (in Beacon.Api / worker)                Service Bus namespace (Standard, private endpoint)
 Tickets ─┐                                      topic: beacon-events
 Knowledge┼─► outbox tables ─► Outbox relay ─►     ├─ sub: notifications (filter: type LIKE 'beacon.ticket.%')
 Billing ─┘   (per module)     (ca-beacon-worker)  ├─ sub: search        (filter: type LIKE 'beacon.article.%' OR ...)
                                                   ├─ sub: webhooks      (sessions on, key = ticketId) ─► beacon-relay
                                                   └─ sub: analytics     ─► Event Hubs (via forwarder) ─► warehouse
```

**Publishing** from the outbox relay, with metadata and trace context:

```csharp
public sealed class ServiceBusOutboxPublisher(ServiceBusSender sender) : IOutboxPublisher
{
    public async Task PublishAsync(IReadOnlyList<OutboxMessage> batch, CancellationToken ct)
    {
        using ServiceBusMessageBatch sbBatch = await sender.CreateMessageBatchAsync(ct);
        foreach (var m in batch)
        {
            var message = new ServiceBusMessage(BinaryData.FromString(m.Payload))
            {
                MessageId = m.Id.ToString(),          // enables broker duplicate detection
                Subject = m.Type,                     // e.g. beacon.ticket.resolved.v1
                ContentType = "application/json",
                SessionId = m.AggregateId,            // per-ticket ordering where sessions are enabled
                CorrelationId = m.CorrelationId,
            };
            message.ApplicationProperties["type"] = m.Type;
            message.ApplicationProperties["source"] = "beacon/tickets";
            if (m.TraceParent is not null)
                message.ApplicationProperties["traceparent"] = m.TraceParent;  // captured when the outbox row was written

            if (!sbBatch.TryAddMessage(message))
                throw new InvalidOperationException($"Message {m.Id} too large for a batch");
        }
        await sender.SendMessagesAsync(sbBatch, ct);
    }
}
```

The outbox row stores the W3C `traceparent` from the originating request, so the trace in Application
Insights (Book IX, Chapter 6) continues from "agent clicks Resolve" through the relay to the email sent
minutes later. The Azure SDK also adds its own diagnostic properties; the important part is that the
link starts from the original request, not from the relay's polling loop.

**Consuming** in the Notifications module:

```csharp
public sealed class NotificationsConsumer(ServiceBusClient client, IServiceScopeFactory scopes,
    ILogger<NotificationsConsumer> log) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        await using var processor = client.CreateProcessor("beacon-events", "notifications",
            new ServiceBusProcessorOptions
            {
                AutoCompleteMessages = false,
                MaxConcurrentCalls = 8,                       // bounded concurrency (bulkhead)
                PrefetchCount = 16,
            });

        processor.ProcessMessageAsync += async args =>
        {
            await using var scope = scopes.CreateAsyncScope();
            var dispatcher = scope.ServiceProvider.GetRequiredService<IntegrationEventDispatcher>();
            try
            {
                await dispatcher.DispatchAsync(InboundMessage.From(args.Message), args.CancellationToken);
                await args.CompleteMessageAsync(args.Message, args.CancellationToken);
            }
            catch (PermanentMessageException ex)              // invalid payload, unknown type, etc.
            {
                await args.DeadLetterMessageAsync(args.Message, ex.Reason, ex.Message, args.CancellationToken);
            }
            // any other exception: not completed → lock expires or is abandoned → redelivered,
            // and after MaxDeliveryCount the broker dead-letters it
        };
        processor.ProcessErrorAsync += args =>
        {
            log.LogError(args.Exception, "Service Bus error in {Source}", args.ErrorSource);
            return Task.CompletedTask;
        };

        await processor.StartProcessingAsync(stoppingToken);
        try { await Task.Delay(Timeout.Infinite, stoppingToken); }
        catch (OperationCanceledException) { }
        await processor.StopProcessingAsync();
    }
}
```

The dispatcher routes by `type`, deserializes to the integration event contract, and calls the handler
inside the inbox transaction from section 5. Operational settings that matter:

- **Duplicate detection** on the topic (a window of minutes, keyed by `MessageId`) suppresses most
  duplicates from relay retries, but handlers stay idempotent because the window is finite.
- **Max delivery count 10**, DLQ depth alert per subscription, and `beacon dlq` commands for inspection
  and replay.
- **Lock duration** longer than the handler's p99, with automatic lock renewal for slow handlers.
- **Scaling**: Container Apps scale the worker on subscription backlog (KEDA's Service Bus scaler, Book
  IX, Chapter 3).
- **Identity**: the worker uses its managed identity with "Azure Service Bus Data Receiver" on its
  subscriptions only; the relay has "Data Sender" on the topic only.

> **🔄 Current (as of October 2026):** In .NET, teams either use the Azure SDK (`Azure.Messaging.ServiceBus`)
> directly, as above, or a messaging framework that adds outbox/inbox, sagas, retries and
> transport abstraction: NServiceBus (commercial), MassTransit (version 8 is open source; the maintainers
> announced commercial licensing from version 9), Wolverine, Rebus, or Dapr's pub/sub building block.
> Frameworks save a lot of plumbing for saga-heavy systems; for a handful of events, direct SDK use with
> a small dispatcher is easy to understand. Check licenses before you commit.

> **🔍 Investigation:** After a deployment, the notifications DLQ suddenly holds 3,000 messages with reason
> `DeserializationFailed`. The Tickets module renamed `resolution` to `resolutionCode` in
> `TicketResolvedIntegrationEvent`. The contract test (Book VII, Chapter 2) wasn't run because the
> events weren't in the OpenAPI document. Fix: restore the old field alongside the new one, replay the
> DLQ, and add the integration event schemas to the contract checks in CI so breaking changes fail the
> build.

---

## 10. What can go wrong

- **Dual writes** (save, then publish) instead of an outbox: lost or phantom events.
- **Non-idempotent consumers**: duplicate emails, double counts, repeated webhooks.
- **Unmonitored DLQs**: messages silently piling up for weeks.
- **Unbounded retries** on poison messages, burning capacity and blocking ordered sessions.
- **Events as remote procedure calls**: "events" that are really commands with a single expected handler
  and a reply the sender waits for. You've built slow, hard-to-debug RPC.
- **Leaking internal models** as event payloads: every refactor breaks subscribers.
- **Event chains nobody can follow**: A triggers B triggers C... Without tracing and a documented flow,
  debugging is archaeology. Use orchestration for real processes.
- **Ignoring ordering**: an old `TicketUpdated` applied after a newer one, overwriting fresh data.
- **Consumer lag without alerts**: "eventually" becomes "hours later" during an incident.
- **Large payloads** in messages (attachments, full documents). Brokers have size limits (Service Bus
  Standard: 256 KB); store the blob and send a reference (the "claim check" pattern).

---

## 11. When not to use it

> **🧭 When not to use it:** Don't introduce a broker when:
> - The caller **needs the result now** to continue (validation, a price, a permission check). Use a
>   direct call.
> - Everything lives in **one process and one database**. Do the work in the transaction, or use the
>   outbox table as a queue; a broker adds infrastructure and failure modes.
> - The team has **no operational capacity** for DLQs, lag alerts and replay tooling. Messaging without
>   operations is a way to lose data quietly.
> - **Strong consistency** is a business requirement for the data involved.

And don't use events to avoid deciding who owns a process. If something must happen as a business
rule, someone should own it explicitly: an orchestrator, a handler with tests, a documented flow.

---

## 12. How an experienced engineer thinks about this

- **Distinguishes commands from events**, because ownership and coupling depend on it.
- **Never dual-writes**: outbox for publishing, inbox (or natural idempotency) for consuming.
- **Assumes duplicates and reordering**, and designs handlers with IDs and versions.
- **Treats events as public contracts**, with versioning, schemas and contract tests.
- **Makes asynchronous flows observable**: trace context in messages, lag metrics, DLQ alerts, and a
  diagram of who consumes what.
- **Prefers orchestration for processes** with several steps, timeouts or compensation.
- **Uses the simplest transport that works**: a table first, a broker when fan-out or cross-service
  delivery needs it, a log when volume or replay needs it.
- **Designs the user experience for eventual consistency**, rather than hoping users won't notice.

---

## 13. Check yourself

**Questions**

1. What's the difference between a command and an event, in meaning and in ownership?
2. Compare queues, topics and logs. When would you use each?
3. Why does at-least-once delivery cause duplicates? List three situations that cause redelivery.
4. What's a poison message, and what should happen to it?
5. Explain the dual-write problem and how the transactional outbox solves it. What guarantee does the
   outbox give, and what doesn't it give?
6. How does an inbox table make a consumer's database effects exactly-once? What about external effects?
7. Compare event notification and event-carried state transfer.
8. What's the difference between event-driven and event-sourced?
9. Compare choreography and orchestration for sagas. When does each fit?
10. How do you design a UI for eventual consistency?

**Exercises**

1. Add Service Bus publishing to Beacon's outbox relay (or use RabbitMQ in Docker Compose locally), with
   trace context propagation. Show one trace from the resolve request to the email handler.
2. Implement the inbox pattern for the Notifications consumer and write a test that delivers the same
   message twice and asserts one notification.
3. Write `beacon dlq list` and `beacon dlq replay --dry-run` in `beacon-tools`.
4. Implement the plan upgrade process manager with a timeout and compensation, and test each path
   (success, failure at each step, timeout, duplicate messages).
5. Add JSON Schemas for Beacon's integration events and a CI check that fails on breaking changes.

**Interview-style questions**

- "How do you reliably publish an event when a database record changes?"
- "Your consumer processed a message twice. Why, and how do you make that safe?"
- "Explain sagas. How do you handle a failure in the middle of one?"
- "When would you choose Kafka over a message queue like Service Bus or RabbitMQ?"
- "How do you guarantee ordering of events for one entity while still scaling consumers?"

---

## 14. Going deeper

- Gregor Hohpe and Bobby Woolf, *Enterprise Integration Patterns* — the vocabulary of messaging, still
  accurate twenty years on; [enterpriseintegrationpatterns.com](https://www.enterpriseintegrationpatterns.com).
- Martin Fowler, ["What do you mean by 'Event-Driven'?"](https://martinfowler.com/articles/201701-event-driven.html)
  — notification, state transfer, event sourcing and CQRS distinguished.
- Chris Richardson, [microservices.io patterns](https://microservices.io/patterns/) — transactional
  outbox, saga, idempotent consumer.
- [Azure Service Bus documentation](https://learn.microsoft.com/azure/service-bus-messaging/) — sessions,
  duplicate detection, dead-lettering.
- [CloudEvents specification](https://cloudevents.io).
- Martin Kleppmann, *Designing Data-Intensive Applications*, chapters on stream processing.

**Next:** [Scalability, Reliability and Observability](05-scalability-reliability-and-observability.md)
asks how much load the system can take, how it fails, and how you'd know.
