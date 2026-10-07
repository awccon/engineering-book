# Distributed Systems

Beacon has been distributed since Book III. The browser talks to the BFF, which talks to the API, which
talks to PostgreSQL, Redis, an identity provider, an email service and an AI model. A worker process
reads the outbox, `beacon-relay` calls customer endpoints, and Azure runs several replicas of
everything. Each arrow is a network call, and each network call can fail in ways a method call can't.

This chapter is about what changes when code runs on more than one machine: **partial failure, time,
consistency, retries and idempotency**. These ideas are some of the most durable in the book. They
haven't changed in decades, and every new platform (serverless, edge, AI agents calling tools) runs
into them again.

---

## 1. The problem: the network is not a method call

Within one process, a method call either returns or throws, quickly and reliably. Over a network, a
request can:

1. succeed, and you get the response;
2. fail before reaching the server (connection refused, DNS failure);
3. reach the server, fail there, and you get an error;
4. reach the server, **succeed**, and the response is lost (timeout, connection reset);
5. be delayed for a long time and arrive after you've given up and retried;
6. be delivered twice, by a proxy or a retry layer you didn't know about.

Cases 4–6 are the hard ones. After a timeout, **you don't know whether the operation happened**. Every
distributed systems technique in this chapter exists to deal with that uncertainty.

The classic list of false assumptions, the **fallacies of distributed computing** (Peter Deutsch and
others at Sun, 1990s):

1. The network is reliable.
2. Latency is zero.
3. Bandwidth is infinite.
4. The network is secure.
5. Topology doesn't change.
6. There is one administrator.
7. Transport cost is zero.
8. The network is homogeneous.

Every one of them has caused a Beacon-shaped outage somewhere. In cloud environments, "topology doesn't
change" is especially wrong: containers are rescheduled, IP addresses change, replicas scale in and out,
and a deployment replaces every instance.

---

## 2. The mental model: partial failure, unbounded delay, no shared clock

A distributed system is a set of processes that communicate by messages, where:

- **Any component can fail independently** while others keep running (partial failure).
- **Messages can be delayed arbitrarily**, so you can't distinguish a slow node from a dead one.
- **There is no global clock.** Each machine's clock drifts, and NTP corrections can move it backwards.

> **🧱 Durable:** In a distributed system, **you can't tell the difference between "slow" and "dead"**.
> All you have is "I haven't heard back yet". A timeout is not a fact about the other side; it's a
> decision you made about how long to wait. Everything else follows from designing around that.

Three consequences shape every design:

1. **Every remote call needs a timeout**, and every timeout creates uncertainty about whether the
   operation happened.
2. **Retries are necessary** to survive transient failures, so **operations must be safe to repeat**
   (idempotent).
3. **Different nodes will see different states** for some time, so you must decide what consistency each
   piece of data actually needs.

---

## 3. Timeouts, deadlines and cancellation

A call without a timeout can hang forever, holding a thread, a connection and memory. Under load, those
hung calls pile up until the caller itself stops responding. That is how one slow dependency takes down
a whole system.

**Choose timeouts from data**, not round numbers. Look at the dependency's latency distribution (Book IX,
Chapter 6): a timeout slightly above its p99.9 lets almost all healthy requests through while bounding
the damage when it degrades. Separate the **connect timeout** (short: seconds) from the **request
timeout** (longer, depends on the operation).

**Propagate deadlines.** If the user's request has a 10-second budget and has already used 7, the
downstream call should get 3, not its own fresh 10. In .NET this is the `CancellationToken`: the
request's `HttpContext.RequestAborted` flows into EF Core, `HttpClient` and the chat client, and linked
tokens add per-call limits:

```csharp
public async Task<TicketSummary> SummarizeAsync(TicketId id, CancellationToken requestAborted)
{
    // Never wait more than 8 s for the model, and never longer than the caller is willing to wait
    using var cts = CancellationTokenSource.CreateLinkedTokenSource(requestAborted);
    cts.CancelAfter(TimeSpan.FromSeconds(8));

    var ticket = await tickets.FindAsync(id, cts.Token);
    var response = await chat.GetResponseAsync(SummaryPrompt.For(ticket), cancellationToken: cts.Token);
    return SummaryParser.Parse(response);
}
```

gRPC propagates deadlines across services automatically; for HTTP, some teams pass a
`Request-Timeout`-style header so the callee can stop work the caller has abandoned.

> **⚠️ What can go wrong:** Timeouts that are **longer downstream than upstream** waste work: the BFF gives
> up after 30 s, but the API keeps waiting 60 s for the database, and the database keeps running the
> query after that. Make timeouts **shrink** as you go deeper, and pass cancellation all the way down so
> abandoned work actually stops (PostgreSQL cancels a query when Npgsql's token fires).

---

## 4. Retries: necessary and dangerous

Most failures in cloud systems are **transient**: a dropped connection, a replica restarting during a
deployment, a brief throttle. Retrying fixes them. But retries also **multiply load at exactly the
moment a system is struggling**, which can turn a brief slowdown into an outage.

Rules for safe retries:

1. **Retry only transient errors**: connection failures, timeouts, 408, 429, 502, 503, 504. Don't retry
   400, 401, 403, 404 or 422; they'll fail again. Treat 500 with care: it may be a bug that will
   always fail.
2. **Retry only idempotent operations** (section 5), or make them idempotent first.
3. **Back off exponentially with jitter**: wait 1 s, 2 s, 4 s... plus a random fraction, so thousands of
   clients don't retry in synchronized waves. (`beacon-relay` in Book XII does exactly this.)
4. **Honor `Retry-After`** when the server sends it (429, 503).
5. **Cap attempts and total time.** A retry policy must fit inside the caller's deadline.
6. **Retry at one layer only.** If the SDK retries 3 times, your `HttpClient` handler retries 3 times,
   and your service retries 3 times, one failing call becomes 27 requests. With three layers of
   services, 729.

```text
Retry amplification with 3 attempts per layer:

 Browser ─► BFF (3) ─► API (3) ─► Search service (3) ─► Database
 1 click  =  3      ×    3     ×        3              = 27 database queries for one failure
```

In .NET, `Microsoft.Extensions.Http.Resilience` (built on Polly) gives you this as configuration:

```csharp
builder.Services.AddHttpClient<IdentityProviderClient>(c => c.BaseAddress = new(idpUrl))
    .AddResilienceHandler("idp", pipeline =>
    {
        pipeline.AddTimeout(TimeSpan.FromSeconds(10));                 // total, including retries
        pipeline.AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            BackoffType = DelayBackoffType.Exponential,
            UseJitter = true,
            Delay = TimeSpan.FromMilliseconds(200),
        });                                                            // retries transient errors + 429, honors Retry-After
        pipeline.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            FailureRatio = 0.5,
            MinimumThroughput = 20,
            SamplingDuration = TimeSpan.FromSeconds(30),
            BreakDuration = TimeSpan.FromSeconds(15),
        });
        pipeline.AddTimeout(TimeSpan.FromSeconds(3));                  // per attempt
    });
```

`AddStandardResilienceHandler()` gives sensible defaults for this whole stack; customize when you know
the dependency.

### Circuit breakers and retry budgets

A **circuit breaker** stops calling a dependency that is failing most requests, fails fast for a while,
then lets a few trial requests through. It protects the dependency (giving it room to recover) and the
caller (no threads stuck waiting). When the circuit is open, the caller needs a **fallback**: cached
data, a degraded response ("AI summary unavailable"), or a clear error.

A **retry budget** limits retries to a fraction of normal traffic (say 10%), so that when everything is
failing, retries can't multiply load. gRPC and some service meshes implement this; you can approximate
it with a rate limiter on the retry path.

### Hedging

For latency-sensitive **reads**, a **hedged request** sends a second copy to another replica if the
first hasn't answered within, say, the p95 latency, and takes whichever answers first. It cuts tail
latency at the cost of a few percent extra load. Polly supports hedging; use it only for idempotent,
cheap reads.

---

## 5. Idempotency: making "maybe it happened" safe

An operation is **idempotent** if doing it twice has the same effect as doing it once. With idempotency,
the answer to "did my timed-out request happen?" stops mattering: just do it again.

Some operations are naturally idempotent:

- `GET`, `PUT` (set to this value), `DELETE` (delete if present).
- "Set status to Resolved" (but not "toggle status").
- `INSERT ... ON CONFLICT DO NOTHING` with a natural key.

Others aren't:

- `POST /api/tickets` (creates a new ticket each time).
- "Add comment", "send email", "charge card", "increment counter".

### Idempotency keys

For non-idempotent operations, the client generates a unique **idempotency key** per logical operation
and sends it with every attempt. The server records the key with the result; a repeat with the same key
returns the stored result instead of acting again.

```http
POST /api/tickets HTTP/1.1
Idempotency-Key: 6f1c2b8e-1d4a-4f0e-9b7a-3c5d2e8f9a01
Content-Type: application/json

{ "title": "Cannot export invoices", "priority": "High" }
```

A Beacon implementation, as an endpoint filter backed by PostgreSQL:

```sql
CREATE TABLE idempotency_keys (
    key          uuid        NOT NULL,
    principal    text        NOT NULL,         -- keys are scoped per caller
    request_hash bytea       NOT NULL,         -- detect key reuse with a different body
    status       smallint,                     -- NULL while in progress
    response     jsonb,
    created_at   timestamptz NOT NULL DEFAULT now(),
    PRIMARY KEY (principal, key)
);
-- a scheduled job deletes rows older than 24 hours
```

```csharp
public sealed class IdempotencyFilter(BeaconDbContext db, ICurrentUser user) : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(EndpointFilterInvocationContext ctx, EndpointFilterDelegate next)
    {
        var http = ctx.HttpContext;
        if (!Guid.TryParse(http.Request.Headers["Idempotency-Key"], out var key))
            return await next(ctx);                                  // optional: or require it for this endpoint

        var hash = await RequestHash.ComputeAsync(http.Request);     // SHA-256 of method, path, body
        var ct = http.RequestAborted;

        // Claim the key atomically. Only one concurrent request can insert it.
        var claimed = await db.Database.ExecuteSqlAsync($"""
            INSERT INTO idempotency_keys (key, principal, request_hash)
            VALUES ({key}, {user.Id}, {hash})
            ON CONFLICT DO NOTHING
            """, ct) == 1;

        if (!claimed)
        {
            var existing = await db.IdempotencyKeys.AsNoTracking()
                .SingleAsync(k => k.Principal == user.Id && k.Key == key, ct);
            if (!existing.RequestHash.SequenceEqual(hash))
                return Results.Problem(statusCode: 422, title: "Idempotency-Key reused with a different request");
            if (existing.Status is null)
                return Results.Problem(statusCode: 409, title: "A request with this key is in progress");
            return Results.Json(existing.Response, statusCode: existing.Status.Value);   // replay
        }

        var result = await next(ctx);
        await StoredResponse.SaveAsync(db, user.Id, key, result, ct);   // status + body
        return result;
    }
}
```

Points that make this correct rather than merely plausible:

- **The claim is atomic** (`INSERT ... ON CONFLICT`), so two concurrent retries can't both run.
- **Keys are scoped per principal**, so one customer can't replay another's response.
- **The request hash** catches clients that reuse a key for a different operation.
- **In-progress requests** get a 409, and the client retries later.
- **If the server crashes** between claiming and saving, the key stays "in progress". A cleanup job
  removes stale claims after a timeout (the operation itself must be safe at that point; ideally the
  claim, the work and the stored response are in **one database transaction**, which is possible here
  because Beacon's ticket creation is a single PostgreSQL transaction).

> **🔄 Current (as of October 2026):** An IETF draft, "The Idempotency-Key HTTP Header Field", standardizes
> the header name and semantics; payment APIs such as Stripe's popularized the pattern. ASP.NET Core has
> no built-in idempotency middleware, so teams write a filter like this or use a library.

### Idempotent consumers

The same idea applies to messages. Message brokers and Beacon's outbox deliver **at least once**, so
handlers must tolerate duplicates. The survey handler in Chapter 1 checks `ExistsForAsync(ticketId)`;
more generally, record processed message IDs in the same transaction as the handler's effects (Chapter 4).

### "Exactly once" is a property of the whole system

You'll hear claims of "exactly-once delivery". Over an unreliable network, a sender can't know whether a
message arrived, so it must either risk losing it (at most once) or risk sending it twice (at least
once). What systems actually offer is **exactly-once processing**: at-least-once delivery combined with
idempotent or deduplicated handling. Kafka's "exactly once semantics" is this, within Kafka's boundary.
The moment a handler calls an external API, idempotency is your job again.

---

## 6. Time and ordering

### Clocks lie

Each machine's clock drifts and is periodically corrected by NTP, sometimes **backwards**. Two machines
can disagree by milliseconds normally and by seconds or more when something goes wrong. Consequences:

- **Don't use wall-clock timestamps to order events from different machines.** "Last write wins" by
  timestamp silently loses writes when clocks disagree.
- **Measure durations with a monotonic clock** (`Stopwatch`, `TimeProvider.GetTimestamp()`), never by
  subtracting `DateTime.UtcNow` values.
- **Generate timestamps in one place** when ordering matters, typically the database (`now()` in
  PostgreSQL, sequence numbers, or a `version` column).

Beacon uses `Ticket.Version` (Book III) for optimistic concurrency rather than "updated_at" comparisons
for exactly this reason: a counter incremented in a transaction has no clock to be wrong.

### Ordering

Within one PostgreSQL database, transactions give you a clear order. Across systems, you have to build
it:

- **Per-entity sequence numbers** (the ticket's `Version`) let consumers ignore stale events: "I've
  applied version 12; this event is version 11, skip it".
- **Partitioned ordering**: message brokers keep order within a partition or session (all events for
  ticket T-42 go to the same partition), not globally (Chapter 4).
- **Logical clocks** (Lamport timestamps, vector clocks) order events by causality without synchronized
  wall clocks. You'll meet them in databases and CRDTs more than in application code.

### Leases and fencing tokens

Beacon's SLA monitor runs on several worker replicas but must run its scan **on only one at a time**. Book
III used a distributed lock. Distributed locks have a subtle failure: a process acquires the lock, pauses
(a long GC pause, a VM migration), its lease expires, another process acquires the lock, and then the
first process wakes up and **still thinks it holds the lock**. Both act.

```text
Worker A: acquire lease (30 s) ── long pause ─────────────── wakes, writes ✗
Worker B:                         lease expired → acquire ── writes
```

The fix is a **fencing token**: a number that increases with every lease grant, sent with every write,
and checked by the resource. A write with an older token is rejected.

```sql
-- the lease row
UPDATE leases SET holder = @me, token = token + 1, expires_at = now() + interval '30 seconds'
WHERE name = 'sla-monitor' AND (expires_at < now() OR holder = @me)
RETURNING token;

-- every write by the job carries its token, and the target checks it
UPDATE sla_scan_state SET last_run = now(), last_token = @token
WHERE name = 'sla-monitor' AND last_token < @token;
```

For many Beacon-style jobs, a simpler answer is to **make the job idempotent** so that occasionally
running it twice is harmless, and use a PostgreSQL advisory lock or `FOR UPDATE SKIP LOCKED` (Book IV,
Chapter 5) for coordination. Use leases with fencing when double execution would cause real harm.

---

## 7. Consistency: what can different readers see?

When data is replicated (a read replica, a cache, a search index, a copy in another module), readers can
see different versions. **Consistency models** describe what's allowed. From strongest to weakest, the
ones you'll reason about in practice:

| Model | Guarantee | Beacon example |
|---|---|---|
| **Linearizable** (strong) | Every read sees the latest committed write, as if there's one copy | Ticket writes on the PostgreSQL primary |
| **Read-your-writes** | A user always sees their own changes | An agent adds a comment and sees it after redirect |
| **Monotonic reads** | A user never sees time go backwards | List doesn't "lose" a ticket on refresh |
| **Causal** | If B was caused by A, everyone sees A before B | A reply never appears before the comment it answers |
| **Eventual** | If writes stop, all copies converge eventually | Search index, dashboard counts, Redis cache |

Most application data needs **less than linearizability** most of the time, and much of it can be
eventual. The trick is to identify the few places where stale reads cause real harm (double-booking,
overselling, security decisions) and keep those strongly consistent.

> **⚠️ What can go wrong:** Read replicas break read-your-writes. If Beacon routes `GET` requests to a
> replica and an agent's `POST` went to the primary, the redirect after saving can show the ticket
> **without** the agent's change, because the replica is 200 ms behind. Common fixes: read from the
> primary for a few seconds after a user writes (sticky session or a "last write" cookie), wait for the
> replica to reach the write's log position, or return the updated resource in the write response so the
> UI doesn't need to re-read (Beacon's API already returns `TicketResponse` from writes).

### CAP and PACELC

The **CAP theorem** says that when a network **partition** separates replicas, a system must choose
between **consistency** (refuse some requests so nobody sees stale data) and **availability** (answer
everything, accepting that some answers may be stale or conflicting). You can't have both during a
partition. "CA" systems don't exist in practice, because partitions aren't optional.

**PACELC** extends it usefully: if there's a **P**artition, choose **A** or **C**; **E**lse (normal
operation), choose between **L**atency and **C**onsistency. Even without failures, strong consistency
across regions costs round trips. That second half is the trade-off you face daily: synchronous
replication to another region adds tens of milliseconds to every write.

For Beacon:

- **PostgreSQL primary with a synchronous standby in another zone** (Book IX): consistency over latency
  within a region; a short write pause during failover.
- **Cross-region replica**: asynchronous, for disaster recovery. Failing over may lose the last few
  seconds of writes (a recovery point objective of seconds, not zero).
- **Search index, caches, AI embeddings**: eventually consistent, with explicit freshness ("indexed 30 s
  ago") where users would notice.

### Consensus: use it, don't build it

Getting several nodes to **agree** on a value (who's the leader, what's the committed log) despite
failures is the **consensus** problem, solved by algorithms like Paxos and Raft. They're subtle, and
implementations take years to get right. You'll use consensus constantly through etcd (in Kubernetes),
ZooKeeper, PostgreSQL high-availability managers like Patroni, CockroachDB, Azure's managed services and
Kafka's KRaft. You should almost never implement it.

> **🧱 Durable:** When you need coordination (leader election, locks, unique sequences, configuration),
> **borrow a system that already solves consensus**, usually your database. A PostgreSQL row lock,
> advisory lock or sequence gives you strong guarantees for free, as long as you stay within one
> database.

---

## 8. Designing for partial failure

Putting the pieces together, a few patterns turn "any dependency can fail" into graceful degradation:

- **Bulkheads**: isolate resources per dependency so one slow dependency can't exhaust everything. Separate
  `HttpClient` connection limits, separate semaphores (`beacon-relay`'s per-endpoint limits in Book XII),
  separate worker pools.
- **Fallbacks and degraded modes**: decide in advance what Beacon does when each dependency is down. AI
  model unavailable → hide the summary panel, triage manually. Search unavailable → fall back to
  PostgreSQL full-text search. Email provider down → queue in the outbox and retry for hours.
- **Load shedding**: when overloaded, reject some work quickly (429/503 with `Retry-After`) rather than
  accept everything and time out on everything. ASP.NET Core rate limiting and concurrency limiters
  (Book III) do this at the edge.
- **Asynchronous boundaries**: if the user doesn't need the result now, accept the request, return 202,
  and process it from a queue. Queues absorb bursts and decouple availability (Chapter 4).
- **Static stability**: design so that when a control plane or dependency fails, the system keeps
  running in its last known good state (cached configuration, cached tokens, cached entitlements)
  rather than failing closed everywhere.

A useful exercise is a **dependency failure table** for each service:

| Dependency | If slow | If down | User impact | Detection |
|---|---|---|---|---|
| PostgreSQL primary | Timeouts after 5 s; shed load | Failover (≤60 s); writes fail meanwhile | Read-only mode banner | DB health check, error SLO |
| Redis (HybridCache) | Skip cache after 100 ms | In-memory L1 only | Slower pages | Cache miss rate |
| Identity provider | Existing sessions continue | New logins fail | Login error page | Synthetic login probe |
| AI model | Summary timeout at 8 s | Feature hidden; manual triage | Less automation | AI error rate, circuit state |
| Email provider | Outbox backlog | Outbox backlog grows; retry | Delayed notifications | Outbox age alert |

Filling in this table often reveals dependencies nobody thought about, and missing timeouts.

---

## 9. In practice: hardening Beacon's ticket creation path

Let's trace `POST /api/tickets` from the browser and fix each distributed-systems issue.

```text
Browser ─► Front Door ─► BFF ─► API ─► PostgreSQL (ticket + outbox, one transaction)
                                  └──► AI triage (synchronous, Book XI)  ← problem
Outbox worker ─► Notifications ─► email provider
              └► beacon-relay ─► customer webhooks
```

**Problem 1: duplicate tickets.** Users double-click; the browser or BFF retries on a network blip; the
API created the ticket, but the response was lost. Fix: the React form generates an idempotency key
when the form is opened (not per click) and sends it with every attempt; the API applies the filter
from section 5. Disable the submit button while pending as a usability measure, not as the guarantee.

```ts
// web/src/features/tickets/NewTicketForm.tsx (excerpt)
const idempotencyKey = useMemo(() => crypto.randomUUID(), []);   // one per form instance
const create = useMutation({
  mutationFn: (input: CreateTicketInput) => ticketsApi.create(input, { idempotencyKey }),
  retry: 2,                                                       // safe now
});
```

**Problem 2: AI triage on the request path.** Book XI added AI triage when a ticket is created. If the
model is slow, ticket creation is slow; if it's down, creation fails. Fix: create the ticket with
default triage, commit, and publish `TicketCreated` through the outbox. A triage handler calls the model
asynchronously and applies the suggestion (with its own idempotency: skip if the ticket already has a
triage result or a human has changed priority). The UI shows "Suggested priority: analyzing…" and
updates via SignalR.

**Problem 3: retries in three layers.** The BFF's YARP proxy, the browser's TanStack Query and the
`HttpClient` in the API all retried. Fix: the BFF proxies **without** retries for non-idempotent
methods; the browser retries (with the idempotency key); the API's outbound calls each have one
resilience pipeline.

**Problem 4: timeouts that grow downstream.** Front Door's origin timeout was 60 s, BFF 100 s (YARP
default), API's database command timeout 30 s, AI client 100 s (`HttpClient` default). Fix: Front Door
30 s > BFF 25 s > API request 20 s > database 5 s for writes. The AI call is no longer on this path.

**Problem 5: notifications after a failover.** During a PostgreSQL failover, the outbox worker had
claimed messages, sent some emails, and lost its connection before marking them sent. After failover it
sent them again. Fix: the email handler records `notification_deliveries(message_id)` and the provider
call carries the message ID as its own idempotency key, so duplicate sends are suppressed at both
layers.

After these changes, the ticket creation path depends synchronously only on PostgreSQL. Everything else
is asynchronous, retried and idempotent.

---

## 10. What can go wrong

- **No timeout**, or the default 100 s `HttpClient` timeout everywhere. One slow dependency exhausts
  connections and threads.
- **Retrying non-idempotent operations**: duplicate tickets, double emails, double charges.
- **Retry storms**: synchronized retries without jitter, or retries at every layer, turning a blip into
  an outage.
- **Treating a timeout as a failure**: showing "failed" and letting the user resubmit when the operation
  actually succeeded.
- **Wall-clock ordering** across machines: lost updates and impossible event sequences.
- **Distributed locks without fencing** for operations where double execution causes harm.
- **Assuming read-your-writes** with replicas or caches.
- **Hidden synchronous dependencies**: a "non-critical" call (analytics, feature flags, AI) on the
  critical path without a timeout or fallback.
- **Ignoring the cost of coordination**: cross-region synchronous calls on every request.

---

## 11. When not to use it

> **🧭 When not to use it:** Don't **add** distribution to escape these problems you don't yet have. One
> process and one database avoid most of this chapter. Keep work in a single PostgreSQL transaction when
> you can, and introduce queues, replicas and separate services only when there's a measured need. And
> don't build your own consensus, distributed lock service or exactly-once delivery: borrow them from
> your database or a managed service.

Also avoid over-engineering the small things: an internal admin tool calling one internal API with sane
timeouts doesn't need hedging, retry budgets and fencing tokens.

---

## 12. How an experienced engineer thinks about this

- **Assumes every remote call can fail, hang or succeed silently**, and asks "what happens if this times
  out after the server acted?"
- **Makes operations idempotent first**, then adds retries.
- **Designs timeouts top-down** from the user's budget, and makes them shrink with depth.
- **Retries in exactly one layer**, with backoff, jitter and a cap.
- **Uses the database for coordination and ordering** rather than clocks or homemade locks.
- **Chooses consistency per piece of data**, and makes staleness visible to users where it matters.
- **Minimizes synchronous dependencies on critical paths**, moving the rest behind queues.
- **Writes down degraded modes** before the incident, not during it.

---

## 13. Check yourself

**Questions**

1. Why can't you distinguish a slow node from a dead one? What does that imply about timeouts?
2. Which HTTP status codes should be retried? Which shouldn't?
3. How does retry amplification happen, and how do you prevent it?
4. What makes an operation idempotent? Give two naturally idempotent and two non-idempotent Beacon
   operations.
5. Walk through how an idempotency key implementation handles two concurrent retries, a retry after
   success, and key reuse with a different body.
6. Why is "exactly-once delivery" impossible, and what do systems offer instead?
7. Why shouldn't you order events from different machines by wall-clock timestamp?
8. What problem do fencing tokens solve?
9. Explain CAP and PACELC with a Beacon example of each trade-off.
10. How do read replicas break read-your-writes, and how do you fix it?

**Exercises**

1. Implement the idempotency filter for `POST /api/tickets` and `POST /api/tickets/{id}/comments`. Write
   a test that fires the same request concurrently 10 times and asserts exactly one ticket exists.
2. Configure the timeouts in Beacon's request path to shrink with depth. Add a test (or a chaos
   experiment with Toxiproxy) that makes PostgreSQL slow and verifies the API returns 503 quickly instead
   of hanging.
3. Fill in a dependency failure table for a system you work on. Find at least one missing timeout or
   fallback.
4. Move AI triage off the ticket creation path, as in section 9, and show the UI updating when triage
   completes.
5. Simulate a GC pause in a lease holder (sleep past the lease) and demonstrate a double execution; then
   add a fencing token and show the stale write is rejected.

**Interview-style questions**

- "What happens when a request to a payment service times out? How do you design the client?"
- "Explain idempotency and how you'd implement it for a POST endpoint."
- "What's the CAP theorem? Is it relevant to your day-to-day work?"
- "How would you prevent a retry storm?"
- "How do you run a scheduled job on exactly one of several instances?"

---

## 14. Going deeper

- Martin Kleppmann (with Chris Riccomini for the second edition), *Designing Data-Intensive Applications*, chapters on replication,
  consistency and "The Trouble with Distributed Systems" — the single best book on these topics.
- Marc Brooker's blog ([brooker.co.za](https://brooker.co.za/blog/)) and the Amazon Builders' Library
  articles on timeouts, retries and backoff with jitter, and on avoiding fallback in distributed systems.
- Martin Kleppmann, ["How to do distributed locking"](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html)
  — fencing tokens explained.
- [Microsoft.Extensions.Http.Resilience](https://learn.microsoft.com/dotnet/core/resilience/http-resilience)
  and [Polly documentation](https://www.pollydocs.org).
- Brandur Leach, "Implementing Stripe-like Idempotency Keys in Postgres" — a detailed walkthrough.
- Diego Ongaro and John Ousterhout, "In Search of an Understandable Consensus Algorithm" (the Raft
  paper) — readable, if you want to know how consensus works.

**Next:** [Messaging and Event-Driven Architecture](04-messaging-and-event-driven-architecture.md) builds
on idempotency and eventual consistency to connect parts of a system through queues, topics and events.
