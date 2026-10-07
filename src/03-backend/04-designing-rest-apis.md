# Designing REST APIs

An API is a contract that outlives the code behind it. Once a mobile app, a partner
integration or another team depends on `/api/tickets`, changing its shape is expensive:
you can't redeploy their code. Good API design decisions (resource names, error formats,
pagination, versioning, concurrency) are cheap on day one and very expensive to retrofit.

This chapter is about designing APIs *before* writing code: the principles of REST, the
practical conventions most teams adopt, and the problems every real API eventually faces.

---

## 1. The problem: a contract between independent parties

An API's consumers:

- deploy on their own schedule (mobile apps stay installed for years),
- read your documentation, not your code,
- retry when networks fail,
- and will depend on any behavior you expose, intended or not (*Hyrum's Law*).

So the design goals are: **predictable** (consistent conventions), **evolvable** (can grow
without breaking clients), **robust** (safe under retries and concurrency) and
**efficient** (no unnecessary round trips or giant payloads).

---

## 2. The mental model: resources and representations

REST (Representational State Transfer), described by Roy Fielding in 2000, models an API
as **resources** (nouns) identified by **URLs**, manipulated through a **uniform
interface** (HTTP methods) by transferring **representations** (usually JSON).

```text
 Resource                URL                          Methods
 ───────────────────     ───────────────────────────  ─────────────────────────
 Collection of tickets   /api/tickets                 GET (list), POST (create)
 One ticket              /api/tickets/T-42            GET, PUT/PATCH, DELETE
 A ticket's comments     /api/tickets/T-42/comments   GET, POST
 One comment             /api/tickets/T-42/comments/7 GET, DELETE
```

Most real-world "REST" APIs follow these principles pragmatically rather than strictly.
(Fielding's full definition includes *hypermedia*: responses containing links that drive
the client's next actions. Few JSON APIs do this fully, and that's fine.)

### Naming conventions

- **Nouns, plural, lowercase, hyphenated**: `/api/tickets`, `/api/knowledge-articles`.
- **No verbs in paths** for CRUD: `POST /api/tickets`, not `POST /api/createTicket`.
- **Nesting expresses ownership**, one or two levels deep at most:
  `/api/tickets/T-42/comments`. For deeper relationships, use top-level resources with
  filters: `/api/comments?ticketId=T-42&author=maria`.
- **Consistent JSON casing**: camelCase is the dominant convention for JSON APIs.

### Actions that aren't CRUD

Business operations don't always map to create/read/update/delete. "Resolve a ticket" is
a state transition with rules, not a field update. Options:

| Approach | Example | Trade-off |
|---|---|---|
| Update a field | `PATCH /tickets/T-42 {"status":"Resolved"}` | Simple, but hides rules; the server must infer intent from field changes |
| Action sub-resource | `POST /tickets/T-42/resolve` | Explicit, maps to domain methods; "not pure REST" |
| Resource for the action | `POST /tickets/T-42/resolutions` | Pure REST; can carry data (resolution notes) |

Beacon uses explicit action endpoints (`/resolve`, `/assign`), because they map directly
to domain operations with rules (Book I, Chapter 3). It's what most large APIs (GitHub,
Stripe) do. Pick a convention and use it everywhere.

---

## 3. Design the API before the code

Writing the contract first, before implementing, forces clarity and allows parallel work
(the frontend can start against a mock).

A lightweight process:

1. **List the use cases** from the consumer's point of view. "An agent views their queue,
   sorted by urgency." "A customer adds a comment."
2. **Identify resources and operations** for each.
3. **Sketch requests and responses** in JSON, including errors.
4. **Review with consumers** (frontend developers, partner teams).
5. **Write it down** as an OpenAPI document or as typed contracts, and implement.

A design sketch for one endpoint:

```text
GET /api/tickets?status=open&assignee=maria&sort=-priority,createdAt&limit=20&cursor=...

200 OK
{
  "items": [
    { "id": "T-42", "title": "VPN drops", "status": "InProgress", "priority": "Urgent",
      "assignee": "maria", "createdAt": "2026-10-07T09:12:00Z", "commentCount": 3 }
  ],
  "nextCursor": "eyJjIjoiMjAyNi0xMC0wN1QwOToxMjowMFoiLCJpIjo0Mn0"
}

400  invalid filter (problem details, with field errors)
401  not authenticated
```

Questions the sketch surfaces early: Can agents see tickets not assigned to them? What's
the default sort? Max page size? Is `assignee` a login or a user ID? Answering these on
paper is much cheaper than in code.

---

## 4. Request and response design

### Consistent shapes

- **Collections** return an object, not a bare array: `{ "items": [...], "nextCursor": ... }`.
  An object can grow (counts, links) without breaking clients; a bare array can't.
- **Single resources** return the representation directly.
- **Creation** returns `201 Created` with a `Location` header and the created resource.
- **IDs** are opaque strings to clients (`"T-42"`), even if they're integers internally.
  This lets you change ID schemes later.
- **Timestamps** in ISO 8601 UTC with offset: `"2026-10-07T09:12:00Z"`.
- **Enums as strings**: `"InProgress"`, not `1`. Numbers break silently when someone
  reorders the enum.
- **Money** as a decimal string or integer minor units plus currency, never floating point.

### Errors

Use **Problem Details** consistently (Chapter 2), with machine-readable details for
validation:

```json
{
  "type": "https://beacon.example.com/problems/validation",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "title": ["Title is required."],
    "priority": ["'Critical' is not a valid priority."]
  },
  "traceId": "00-..."
}
```

Clients can display field errors next to form inputs, and support can find the request
by `traceId`.

---

## 5. Pagination, filtering and sorting

Any collection that can grow must be paginated from day one. Retrofitting pagination is a
breaking change.

### Offset pagination

```text
GET /api/tickets?page=3&pageSize=20       → OFFSET 40 LIMIT 20
```

Simple and supports "jump to page 7." But the database must skip all previous rows (slow
at high offsets), and results shift if rows are inserted or deleted while paging
(duplicates or missed items).

### Cursor (keyset) pagination

```text
GET /api/tickets?limit=20
→ { items: [...], nextCursor: "..." }      cursor encodes the last item's sort key, e.g. (createdAt, id)
GET /api/tickets?limit=20&cursor=...       → WHERE (created_at, id) < (@c, @i) ORDER BY created_at DESC, id DESC LIMIT 20
```

Fast at any depth (uses an index, Book IV), and stable under inserts. But no "jump to
page N" and no total count for free.

| | Offset | Cursor |
|---|---|---|
| Deep pages | Slow | Fast |
| Concurrent inserts | Duplicates / skips | Stable |
| Jump to page N | Yes | No |
| Best for | Admin tables, small datasets | Feeds, infinite scroll, large datasets, APIs |

Make cursors **opaque** (base64-encoded JSON) so you can change their contents later.

### Filtering and sorting

- Filters as query parameters: `?status=open&assignee=maria&createdAfter=2026-10-01`.
- Sorting: `?sort=-priority,createdAt` (minus for descending) is a common convention.
- **Allow-list** sortable and filterable fields. Never pass client input directly into SQL
  `ORDER BY` (Chapter 9).
- **Cap page sizes** (`limit` max 100), or a single request can ask for a million rows.

---

## 6. Idempotency for unsafe operations

Chapter 1 explained that `POST` isn't idempotent: if a "create ticket" request times out,
the client can't know whether it succeeded, and a retry might create a duplicate.

The standard solution is an **idempotency key**: the client generates a unique key per
logical operation and sends it in a header. The server stores the key with the result; a
repeat request with the same key returns the stored result instead of performing the
operation again.

```http
POST /api/tickets
Idempotency-Key: 7c9e6679-7425-40de-944b-e07fc1f90ae7
Content-Type: application/json

{ "title": "Cannot log in", "priority": "High" }
```

```text
 first request  ─► key unseen ─► create ticket ─► store (key → 201 + body) ─► 201
 retry          ─► key seen   ─► return stored response                     ─► 201 (same ticket)
 same key, different body ─► 422: key reused with a different request
```

Implementation needs care: store the key and the result atomically with the operation
(in the same database transaction), expire keys after a period (24 hours is common), and
handle two concurrent requests with the same key (a unique constraint makes the second
wait or fail). Stripe's API popularized this pattern; it's essential for payments and
valuable for any "create" that clients might retry.

---

## 7. Concurrency: lost updates and ETags

Two agents open ticket T-42. Both edit the title. Agent A saves; agent B saves a moment
later, overwriting A's change without knowing it existed. This is the **lost update**
problem.

**Optimistic concurrency** with ETags (Chapter 1) solves it:

```text
GET /api/tickets/T-42            → 200, ETag: "5"     (version 5)
PUT /api/tickets/T-42            If-Match: "5"  → 200, ETag: "6"     (A succeeds)
PUT /api/tickets/T-42            If-Match: "5"  → 412 Precondition Failed (B must reload)
```

The server compares the version the client saw with the current one. Book IV implements
the version with a PostgreSQL row version column and EF Core's concurrency tokens.

---

## 8. Versioning and evolution

APIs must change. The goal is to change them **without breaking existing clients**.

### Non-breaking (additive) changes

Safe if clients follow the *tolerant reader* principle (ignore unknown fields):

- adding optional request fields,
- adding response fields,
- adding new endpoints,
- adding new enum values (*only* if clients handle unknown values; document this upfront).

### Breaking changes

- removing or renaming fields or endpoints,
- changing types or formats (`"id": 42` → `"id": "T-42"`),
- making optional fields required,
- changing semantics (status code meanings, default sorts, validation rules).

### Versioning strategies

| Strategy | Example | Notes |
|---|---|---|
| URL path | `/api/v2/tickets` | Most common; obvious; easy to route and cache |
| Query string | `/api/tickets?api-version=2.0` | Used by Azure APIs |
| Header | `Api-Version: 2` | Clean URLs; less visible |
| Media type | `Accept: application/vnd.beacon.v2+json` | Most "RESTful"; least convenient |

For ASP.NET Core, the `Asp.Versioning.Http` package supports all of these.

> **🧭 When not to version:** A new version is a big commitment: you must run, document,
> secure and eventually retire both versions. Prefer additive evolution and only create a
> new major version for genuinely breaking redesigns. For an API used only by your own
> frontend deployed together with the backend, you may not need versioning at all.

When you do retire a version, communicate early, add `Deprecation` and `Sunset` headers,
monitor who still calls it, and give a generous timeline.

---

## 9. REST isn't the only option

| Style | Strengths | Weaknesses | Good for |
|---|---|---|---|
| **REST/JSON** | Universal, cacheable, simple tooling | Over/under-fetching; many round trips for complex views | Public and most internal APIs |
| **GraphQL** | Clients request exactly the fields they need in one query | Caching is harder; complex authorization and performance (N+1) | Many client types with varied data needs |
| **gRPC** | Fast binary protocol (HTTP/2 + Protobuf), streaming, strict contracts | Not browser-native; less human-readable | Service-to-service communication |
| **Async messaging** | Decoupled, resilient | Eventual consistency, harder to debug | Events between services (Book XIII) |

> **🧭 When not to use REST:** If you have many different clients (web, mobile, partners)
> each needing very different slices of a large, interconnected graph of data, GraphQL can
> save a lot of endpoint proliferation. For high-volume internal service-to-service calls,
> gRPC's performance and contracts are compelling. For everything else, REST's
> simplicity usually wins.

---

## 10. In practice: Beacon's API design

Here's Beacon's API surface for tickets, designed before extending the implementation:

```text
GET    /api/tickets                       list (filters: status, assignee, priority; cursor pagination)
POST   /api/tickets                       create (Idempotency-Key supported)
GET    /api/tickets/{id}                  get one (ETag)
PATCH  /api/tickets/{id}                  edit title/description (If-Match required)
POST   /api/tickets/{id}/assign           { "assignee": "maria" }
POST   /api/tickets/{id}/resolve
POST   /api/tickets/{id}/reopen
GET    /api/tickets/{id}/comments
POST   /api/tickets/{id}/comments
```

### Cursor pagination

```csharp
// src/Beacon.Api/Common/Paging.cs
using System.Buffers.Text;
using System.Text.Json;

namespace Beacon.Api.Common;

public sealed record Page<T>(IReadOnlyList<T> Items, string? NextCursor);

public readonly record struct TicketCursor(DateTimeOffset CreatedAt, int Id)
{
    public string Encode() => Base64Url.EncodeToString(JsonSerializer.SerializeToUtf8Bytes(this));

    public static TicketCursor? Decode(string? value)
    {
        if (string.IsNullOrEmpty(value)) return null;
        try { return JsonSerializer.Deserialize<TicketCursor>(Base64Url.DecodeFromChars(value)); }
        catch (Exception e) when (e is FormatException or JsonException) { return null; }
    }
}
```

```csharp
// TicketEndpoints.cs (list endpoint)
public sealed record ListTicketsQuery(TicketStatus? Status, string? Assignee, int Limit = 20, string? Cursor = null);

private static Results<Ok<Page<TicketResponse>>, ValidationProblem> List(
    [AsParameters] ListTicketsQuery q, InMemoryTicketRepository repo)
{
    if (q.Limit is < 1 or > 100)
        return TypedResults.ValidationProblem(new Dictionary<string, string[]>
            { ["limit"] = ["Limit must be between 1 and 100."] });

    var cursor = TicketCursor.Decode(q.Cursor);
    var query = repo.All()
        .Where(t => q.Status is null || t.Status == q.Status)
        .Where(t => q.Assignee is null || string.Equals(t.Assignee, q.Assignee, StringComparison.OrdinalIgnoreCase))
        .OrderByDescending(t => t.CreatedAt).ThenByDescending(t => t.Id.Value)
        .Where(t => cursor is null
            || t.CreatedAt < cursor.Value.CreatedAt
            || (t.CreatedAt == cursor.Value.CreatedAt && t.Id.Value < cursor.Value.Id));

    var items = query.Take(q.Limit + 1).ToList();            // fetch one extra to know if there's more
    var hasMore = items.Count > q.Limit;
    if (hasMore) items.RemoveAt(items.Count - 1);

    var next = hasMore ? new TicketCursor(items[^1].CreatedAt, items[^1].Id.Value).Encode() : null;
    return TypedResults.Ok(new Page<TicketResponse>(items.Select(TicketResponse.From).ToList(), next));
}
```

The "fetch one extra" trick tells us whether a next page exists without a separate count
query. In Book IV, this same logic becomes an indexed SQL query.

(`Base64Url` is in `System.Buffers.Text`, available since .NET 9.)

### ETags on GET, `If-Match` on PATCH

The domain gets a `Version` counter that increments on every change (Book IV maps it to a
database concurrency token):

```csharp
// In Ticket: public int Version { get; private set; }  — incremented in every mutating method
```

```csharp
private static async Task<IResult> Get(TicketId id, ITicketRepository repo, HttpContext http, CancellationToken ct)
{
    var ticket = await repo.FindAsync(id, ct);
    if (ticket is null) return TypedResults.NotFound();

    var etag = $"\"{ticket.Version}\"";
    if (http.Request.Headers.IfNoneMatch == etag) return TypedResults.StatusCode(StatusCodes.Status304NotModified);

    http.Response.Headers.ETag = etag;
    return TypedResults.Ok(TicketResponse.From(ticket));
}
```

The `PATCH` endpoint requires `If-Match` and returns `428 Precondition Required` if it's
missing, and `412` if it doesn't match the current version. Implementing it is exercise 2.

---

## 11. What can go wrong

- **Unpaginated collections** that are fine in testing and time out in production.
- **Breaking changes shipped as "small fixes"**: renaming a field, changing a default sort.
- **Leaking internal models**: database columns and domain internals appearing in JSON.
- **Inconsistent conventions** across endpoints (camelCase here, snake_case there;
  different error formats).
- **Chatty APIs** requiring ten calls to render one screen.
- **Retries creating duplicates** without idempotency keys.
- **Lost updates** without concurrency control.
- **Unbounded inputs**: no max page size, no max string lengths.

---

## 12. How an experienced engineer thinks about this

- **Design from the consumer's use cases**, not from the database schema.
- **Make the contract explicit and review it** before implementing.
- **Plan for evolution**: wrapper objects, string enums, opaque IDs and cursors, tolerant
  readers. These cost nothing now and save versions later.
- **Assume retries and concurrency.** Idempotency keys and ETags are part of a robust API,
  not extras.
- **Consistency beats perfection.** A consistent, slightly imperfect convention is better
  than a mix of ideal ones.

---

## 13. Check yourself

**Questions**

1. How would you model "resolve a ticket" in a REST API? What are the trade-offs?
2. Why should collection endpoints return an object rather than an array?
3. Compare offset and cursor pagination.
4. How do idempotency keys work? What must be stored, and when?
5. How do ETags prevent lost updates?
6. Which changes are breaking? Is adding an enum value breaking?
7. When would you choose GraphQL or gRPC over REST?

**Exercises**

1. Write an OpenAPI-style design (endpoints, request/response examples, errors) for
   Beacon's knowledge-base articles: create, list with search, publish, archive.
2. Implement `PATCH /api/tickets/{id}` with `If-Match`, returning 428, 412 or 200.
3. Implement idempotency keys for `POST /api/tickets` using an in-memory store keyed by
   the header, including the "same key, different body" case.
4. Review a public API you use (GitHub, Stripe) for its conventions on pagination, errors
   and versioning.

**Interview-style questions**

- "Design an API for a ticketing system."
- "How would you paginate a large dataset in an API?"
- "How do you make a POST endpoint safe to retry?"
- "How do you version an API? How do you avoid needing to?"

---

## 14. Going deeper

- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)
- [Zalando RESTful API Guidelines](https://opensource.zalando.com/restful-api-guidelines/)
- [Stripe API reference](https://docs.stripe.com/api) — a widely admired real-world design
  (see its idempotency and pagination docs).
- Arnaud Lauret, *The Design of Web APIs*.

**Next:** [Chapter 5 — Validation and Serialization](05-validation-and-serialization.md)
covers how requests become trusted, typed objects, and how objects become JSON.
