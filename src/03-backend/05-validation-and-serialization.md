# Validation and Serialization

Every request that reaches your API is untrusted input: possibly malformed, possibly
malicious, often just wrong. Before it can touch your domain, it must be **deserialized**
(JSON → objects) and **validated** (is this acceptable?). On the way out, objects are
**serialized** back to JSON, and the shape of that JSON is your public contract.

These steps sound mechanical, but they're where many subtle bugs live: enums that
serialize as numbers, dates that lose their time zone, nulls that mean "clear this field"
in one place and "don't change it" in another, validation duplicated in five places and
missing from a sixth.

---

## 1. The problem: the boundary between outside and inside

```text
  Outside world (untrusted)           Boundary                     Inside (trusted)
  ─────────────────────────           ──────────────────           ────────────────────
  bytes: {"title":"", ...}  ─► deserialize ─► validate ─► map ─►   domain: Ticket (invariants hold)
  bytes ◄─ serialize ◄─ map ◄──────────────────────────────────    domain objects
```

The boundary has three jobs:

1. **Parse**: turn bytes into typed objects, rejecting what can't be parsed.
2. **Validate**: reject well-formed input that breaks the rules of *this request*.
3. **Map**: convert between the external contract and internal domain types.

Doing these deliberately, in one place, keeps the domain clean and the API predictable.

---

## 2. The mental model: layers of validation

Not all validation is the same. There are three layers, and each belongs in a different
place:

| Layer | Question | Example | Where |
|---|---|---|---|
| **Syntactic** | Is it well-formed? | Valid JSON; `priority` is a known enum value; dates parse | Deserializer / binding |
| **Input (request) rules** | Is this request acceptable? | Title 1–200 chars; page size ≤ 100; email format | API boundary: validators |
| **Domain invariants** | Is this allowed in the current state? | Closed tickets can't be commented on; no duplicate assignment | Domain objects and services |

A common mistake is putting all three in one place: either a huge validator that queries
the database (mixing input rules with domain rules), or a domain model full of
string-length checks for a specific form.

> **🧱 Durable:** Validate **shape** at the edge, enforce **rules** in the domain. Edge
> validation gives good error messages fast; domain invariants guarantee correctness no
> matter which entry point (API, CLI, background job) is used.

---

## 3. Serialization with System.Text.Json

ASP.NET Core uses `System.Text.Json` (STJ) by default. It's fast, low-allocation and
secure by default, and somewhat stricter than the older Newtonsoft.Json.

### Defaults in ASP.NET Core

The web defaults (`JsonSerializerDefaults.Web`) use:

- **camelCase** property names,
- **case-insensitive** property matching when reading,
- numbers can be read from strings.

### Common configuration

```csharp
builder.Services.ConfigureHttpJsonOptions(o =>
{
    o.SerializerOptions.Converters.Add(new JsonStringEnumConverter());   // enums as "InProgress"
    o.SerializerOptions.DefaultIgnoreCondition = JsonIgnoreCondition.WhenWritingNull;  // optional
    o.SerializerOptions.UnmappedMemberHandling = JsonUnmappedMemberHandling.Disallow;  // reject unknown fields
});
```

Each choice is a trade-off:

- **String enums** are robust to reordering and readable; numbers break silently.
- **Omitting nulls** shrinks payloads, but some clients can't distinguish "absent" from
  "null." Pick one convention and document it.
- **Disallowing unknown members** catches client typos (`"tittle"`) instead of silently
  ignoring them. Great for requests, but don't make *clients* strict about *your*
  responses, or adding a field breaks them.

### Dates and times

- Use **`DateTimeOffset`** for points in time. It serializes with its offset
  (`2026-10-07T09:12:00+00:00`), so there's no ambiguity.
- `DateTime` with `Kind = Unspecified` is a trap: it serializes without an offset, and
  every client guesses differently.
- **`DateOnly`** and **`TimeOnly`** for calendar dates ("due date") and times of day
  ("office opens at 09:00") that aren't instants.
- Store and transmit UTC; convert to local time only for display.

### Custom converters

When the default representation isn't what you want, write a converter. Beacon's
`TicketId` serializes as `{"value":7}` by default; the API contract wants `"T-7"`:

```csharp
// src/Beacon.Api/Common/TicketIdJsonConverter.cs
using System.Text.Json;
using System.Text.Json.Serialization;
using Beacon.Core.Tickets;

namespace Beacon.Api.Common;

public sealed class TicketIdJsonConverter : JsonConverter<TicketId>
{
    public override TicketId Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
    {
        var text = reader.GetString();
        return TicketId.TryParse(text, null, out var id)
            ? id
            : throw new JsonException($"'{text}' is not a valid ticket id.");
    }

    public override void Write(Utf8JsonWriter writer, TicketId value, JsonSerializerOptions options)
        => writer.WriteStringValue(value.ToString());
}
```

A `JsonException` thrown during deserialization becomes a `400 Bad Request` automatically,
which is the syntactic layer doing its job.

### Polymorphism

Serializing a type hierarchy (different kinds of events, notification channels) needs the
JSON to carry a type discriminator. STJ supports this declaratively:

```csharp
[JsonPolymorphic(TypeDiscriminatorPropertyName = "type")]
[JsonDerivedType(typeof(TicketAssignedDto), "ticketAssigned")]
[JsonDerivedType(typeof(TicketResolvedDto), "ticketResolved")]
public abstract record TicketEventDto(string TicketId, DateTimeOffset OccurredAt);
```

> **⚠️ What can go wrong:** Never deserialize types named *by the payload* (like
> Newtonsoft's `TypeNameHandling.All`). An attacker who controls the type name can make
> your server instantiate dangerous types: a classic remote code execution vulnerability.
> Use an explicit allow-list of derived types, as above.

---

## 4. Contracts vs domain models

Chapter 2 introduced separate request/response types. Here's why it matters so much:

| If you expose domain entities directly... | With dedicated contracts... |
|---|---|
| Renaming a domain property breaks clients | Domain and contract evolve independently |
| Internal fields leak (`Version`, `DomainEvents`, password hashes) | You choose exactly what's exposed |
| Clients can **over-post**: set fields they shouldn't (`"status":"Closed"` on create) | Requests contain only allowed inputs |
| Serialization quirks (cycles, lazy-loaded properties) appear | Contracts are plain, serializable records |

**Over-posting** (also called *mass assignment*) deserves emphasis. If `POST /tickets`
binds directly to an entity with a public `IsAdminOnly` or `Status` setter, any client can
set it. A `CreateTicketRequest(string Title, TicketPriority Priority)` simply has no such
field.

### Mapping

Mapping contracts to and from domain types can be:

- **Manual**: static `From` methods or constructors (what Beacon does). Explicit, fast,
  refactor-safe, and the compiler tells you when a property is added.
- **Source-generated mappers** (Mapperly): generated at compile time, no reflection.
- **Reflection-based mappers** (AutoMapper): convenient for large, similar models, but
  hide behavior, fail at run time and can silently map the wrong things.

> **🧭 When not to use a mapping library:** For most APIs, manual mapping is a few lines
> per type, and those lines are where you make deliberate decisions about what's exposed.
> A mapping library pays off only when you have many large, nearly identical models.

---

## 5. Validation approaches in ASP.NET Core

### Data annotations

```csharp
public sealed record CreateTicketRequest(
    [property: Required, StringLength(200, MinimumLength = 1)] string Title,
    [property: EnumDataType(typeof(TicketPriority))] TicketPriority Priority,
    [property: StringLength(10_000)] string? Description);
```

> **🔄 Current (as of October 2026):** Since .NET 10, minimal APIs validate data
> annotations automatically after calling `builder.Services.AddValidation()`, returning a
> `400` validation problem response before your handler runs. Controllers with
> `[ApiController]` have done this for years.

Data annotations are simple and fine for straightforward rules. They get awkward for
conditional rules ("description required when priority is Urgent") and rules needing
services.

### FluentValidation

A popular library expressing rules in code:

```csharp
public sealed class CreateTicketValidator : AbstractValidator<CreateTicketRequest>
{
    public CreateTicketValidator()
    {
        RuleFor(x => x.Title).NotEmpty().MaximumLength(200);
        RuleFor(x => x.Priority).IsInEnum();
        RuleFor(x => x.Description)
            .NotEmpty().When(x => x.Priority == TicketPriority.Urgent)
            .WithMessage("Urgent tickets need a description so on-call can act on them.");
    }
}
```

It's testable, expressive and composable. Run it from an endpoint filter so handlers stay
clean.

### Validation that needs data

"Assignee must be an existing agent" requires a lookup. That's closer to a domain rule than
input validation: put it in the application service (which returns `Error.Validation` or
`Error.NotFound`), not in a validator that secretly queries the database.

### Consistent error output

Whatever mechanism you use, the response should be the same `ValidationProblemDetails`
shape (Chapter 4), with field names matching the JSON property names the client sent
(camelCase), so clients can map errors to inputs.

---

## 6. Partial updates: the null problem

`PATCH` requests carry only the fields to change. With plain records, you can't tell the
difference between:

```json
{ "description": null }      // clear the description
{ }                          // don't touch the description
```

Both deserialize to `Description = null`. Options:

1. **JSON Merge Patch** (RFC 7396) semantics with a type that tracks presence, e.g. an
   `Optional<T>` wrapper with a custom converter, so "absent" and "null" differ.
2. **JSON Patch** (RFC 6902): a list of operations (`[{"op":"replace","path":"/title","value":"X"}]`).
   Powerful, but verbose for clients, and you must restrict which paths are allowed.
3. **Avoid generic patching**: specific action endpoints (`/assign`, `/resolve`) plus a
   simple `PATCH` for a few editable fields where `null` is never a meaningful value.

Beacon takes option 3, plus a small `Optional<T>` for the one field (description) where
clearing is meaningful:

```csharp
public readonly struct Optional<T>
{
    public Optional(T? value) { HasValue = true; Value = value; }
    public bool HasValue { get; }
    public T? Value { get; }
}

public sealed record UpdateTicketRequest(string? Title, Optional<string?> Description);
```

With a converter that sets `HasValue = true` whenever the property is present in the JSON
(even as `null`), the handler can distinguish the three cases. (STJ only calls a property's
converter when the property appears in the payload, which makes this straightforward.)

---

## 7. In practice: Beacon's request pipeline

### Register converters and validation

```csharp
// Program.cs (excerpt)
builder.Services.ConfigureHttpJsonOptions(o =>
{
    o.SerializerOptions.Converters.Add(new JsonStringEnumConverter());
    o.SerializerOptions.Converters.Add(new TicketIdJsonConverter());
    o.SerializerOptions.UnmappedMemberHandling = JsonUnmappedMemberHandling.Disallow;
});
builder.Services.AddValidation();   // .NET 10: data annotation validation for minimal APIs
```

### Contracts with annotations

```csharp
// src/Beacon.Api/Tickets/TicketContracts.cs
public sealed record CreateTicketRequest(
    [property: Required, StringLength(200, MinimumLength = 1)] string Title,
    [property: EnumDataType(typeof(TicketPriority))] TicketPriority Priority = TicketPriority.Normal,
    [property: StringLength(10_000)] string? Description = null);

public sealed record AddCommentRequest(
    [property: Required, StringLength(100)] string Author,
    [property: Required, StringLength(5_000, MinimumLength = 1)] string Body);

public sealed record TicketResponse(
    TicketId Id, string Title, string? Description, TicketStatus Status, TicketPriority Priority,
    string? Assignee, DateTimeOffset CreatedAt, int CommentCount, int Version)
{
    public static TicketResponse From(Ticket t) => new(
        t.Id, t.Title, t.Description, t.Status, t.Priority, t.Assignee, t.CreatedAt, t.Comments.Count, t.Version);
}
```

### What happens to bad requests now

| Request | Layer that rejects it | Response |
|---|---|---|
| Body isn't JSON | Deserializer | 400 |
| `"priority": "Critical"` | Enum converter (syntactic) | 400 |
| `"tittle": "x"` | Unmapped member handling | 400 |
| `"title": ""` | Data annotations (input rules) | 400 with `errors.title` |
| Comment on closed ticket | `TicketService` (domain rule) | 409 |
| Comment on unknown ticket | `TicketService` | 404 |

Every layer does one job, and the client gets a precise, consistent answer.

### A test for the boundary

```csharp
[Fact]
public async Task Empty_title_returns_field_error()
{
    var response = await _client.PostAsJsonAsync("/api/tickets", new { title = "", priority = "High" });

    Assert.Equal(HttpStatusCode.BadRequest, response.StatusCode);
    var problem = await response.Content.ReadFromJsonAsync<HttpValidationProblemDetails>();
    Assert.Contains("Title", problem!.Errors.Keys, StringComparer.OrdinalIgnoreCase);
}
```

---

## 8. What can go wrong

- **Over-posting** through entities bound directly from requests.
- **Enums as integers** silently changing meaning when reordered.
- **Ambiguous dates** (`DateTime` without kind or offset).
- **Validation only in the frontend.** Anyone can call the API directly.
- **Validation only at the edge**, so other entry points (background jobs, CLI, imports)
  bypass the rules. Domain invariants catch this.
- **Inconsistent error formats** across endpoints.
- **Unsafe polymorphic deserialization** (type names from the payload).
- **Huge payloads**: no limits on string lengths, array sizes or body size. Kestrel limits
  request bodies to ~28.6 MB by default; set limits appropriate for each endpoint.
- **Null ambiguity in PATCH.**

---

## 9. How an experienced engineer thinks about this

- **Parse, don't validate.** Turn untrusted input into types that *can't* be invalid
  (`TicketId`, enums, value objects) as early as possible, so downstream code never
  re-checks.
- **Each validation layer has one job.** Syntax at the deserializer, request rules at the
  edge, invariants in the domain.
- **Contracts are deliberate.** Every exposed field is a promise.
- **Make errors useful** to the client developer: which field, what's wrong, what's allowed.

---

## 10. Check yourself

**Questions**

1. Name the three layers of validation and where each belongs.
2. Why serialize enums as strings?
3. What is over-posting, and how do request contracts prevent it?
4. Why is `DateTimeOffset` usually better than `DateTime` in APIs?
5. Why is deserializing type names from a payload dangerous?
6. How can you distinguish "absent" from "null" in a PATCH request?

**Exercises**

1. Implement the `Optional<T>` JSON converter and use it in `PATCH /api/tickets/{id}` for
   the description field. Test all three cases.
2. Add FluentValidation for `CreateTicketRequest` with the "urgent requires description"
   rule, run from an endpoint filter.
3. Write a test proving that `"status": "Closed"` in a create request is rejected (unknown
   member), not silently applied.
4. Serialize a `DateTime` with each `DateTimeKind` and compare the JSON output.

**Interview-style questions**

- "Where should validation logic live in a layered application?"
- "What's the difference between DTOs and domain entities? Why separate them?"
- "How would you implement partial updates in an API?"

---

## 11. Going deeper

- [Microsoft docs: System.Text.Json overview](https://learn.microsoft.com/dotnet/standard/serialization/system-text-json/overview)
- [Microsoft docs: Model validation](https://learn.microsoft.com/aspnet/core/mvc/models/validation)
- Alexis King, [Parse, don't validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)
- [OWASP: Mass Assignment Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html)

**Next:** [Chapter 6 — Authentication and Authorization](06-authentication-and-authorization.md)
answers "who is calling, and what are they allowed to do?"
