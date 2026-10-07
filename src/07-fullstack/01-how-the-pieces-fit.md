# How the Pieces Fit

The previous six books built Beacon one layer at a time: a C# domain, an ASP.NET Core API,
a PostgreSQL database, a TypeScript and React frontend, and a BFF between them. Each layer
was designed carefully on its own. But users don't experience layers; they experience
**requests that cross all of them**. A click in the browser travels through the SPA, the
BFF, the API, the domain, EF Core and PostgreSQL, and back, in a few hundred milliseconds.
When it's slow or wrong, the cause could be in any of them.

This book is about the system as a whole. This chapter traces a request end to end, asks
where each kind of logic should live, and looks at the trade-offs that only appear when
you see all the pieces together.

---

## 1. The problem: seams between layers

Most full-stack bugs and design problems live at the **seams**:

- The frontend and API disagree about a field's name or nullability.
- Validation exists in the form but not the API (or vice versa), with different rules.
- A screen needs data from five endpoints, so it's slow.
- The same business rule (SLA targets) is implemented in C#, SQL and TypeScript, and they
  drift.
- An error at the database becomes an unhelpful "Something went wrong" in the browser.
- A deployment updates the API before the frontend that depends on the new field.

Each layer can be well built and the system still be fragile. Full-stack engineering is
about designing the seams.

---

## 2. The mental model: one request, end to end

Here is "add a comment to ticket T-42," traced through Beacon:

```text
 Browser                       BFF (ASP.NET Core)            Beacon.Api                    PostgreSQL
 ───────                       ──────────────────            ──────────                    ──────────
 1. user submits form
 2. React Hook Form validates
    (Zod schema)
 3. useMutation → fetch
    POST /api/tickets/T-42/comments
    Cookie: __Host-beacon=…
    X-CSRF: 1            ───────►
                               4. cookie auth → user
                               5. CSRF header check
                               6. YARP adds Bearer token ───► 7. JWT validation
                                                              8. route + bind TicketId
                                                              9. data annotation validation
                                                              10. TicketService.AddCommentAsync
                                                                  ├─ repo.FindAsync ─────────► 11. SELECT … FROM tickets
                                                                  │                                 JOIN comments …
                                                                  ├─ authorization (team)
                                                                  ├─ ticket.AddComment()  (domain invariants, event)
                                                                  └─ SaveChangesAsync ────────► 12. BEGIN; INSERT comment;
                                                                     (outbox interceptor)            UPDATE tickets SET version…
                                                                                                     INSERT outbox; COMMIT
                                                              13. 200 + TicketResponse (JSON)
                               14. proxied response ◄────────
 15. setQueryData, invalidate
     lists → re-render
                                                              16. outbox worker → SignalR ───► other agents' browsers
                                                                                                 update live
```

Sixteen steps, four processes, and at least six places where the request can be rejected
(form validation, CSRF, token validation, binding, input validation, domain rules,
authorization, concurrency). Each check has a purpose: the early ones give fast feedback,
the later ones guarantee correctness.

### Where time goes

A typical breakdown for this request on a healthy system:

| Segment | Typical time |
|---|---|
| Browser → BFF (network + TLS on an existing connection) | 20–80 ms (user's network) |
| BFF → API (same data center) | 1–2 ms |
| API processing (binding, validation, domain) | < 1 ms |
| Database queries and commit | 2–10 ms |
| Response back to browser | 20–80 ms |
| React update and paint | 5–20 ms |

The user's network round trip dominates. That has a clear design consequence: **the number
of round trips from the browser matters more than almost anything on the server.** A screen
that makes five sequential requests pays five network round trips.

> **🧱 Durable:** In distributed systems, latency is dominated by round trips, not
> computation. Batch, parallelize or eliminate them: in the browser (fewer requests),
> between services (fewer calls), and to the database (no N+1).

---

## 3. Where should logic live?

Every rule in Beacon could, in principle, live in the browser, the BFF, the API, or the
database. A practical placement guide:

| Kind of logic | Best home | Why |
|---|---|---|
| **Security decisions** (authentication, authorization) | **API** (and database constraints/RLS as backup) | The client can't be trusted; everything else is UX |
| **Business invariants** (a closed ticket can't be commented on) | **Domain model in the API** | One authoritative implementation; every entry point uses it |
| **Data integrity** (uniqueness, references, valid states) | **Database constraints** | Holds even for scripts, bugs and other services |
| **Input validation** (lengths, formats, required) | **API** (authoritative) + **client** (fast feedback) | Duplicated deliberately, ideally from a shared source |
| **Presentation logic** (formatting, sorting a visible list, what to show) | **Client** | Belongs to the UI; changes with design |
| **Aggregation for reports** | **Database** (SQL) | Set-based work where the data lives (Book IV, Chapter 2) |
| **Session and token handling** | **BFF** | Keeps secrets out of the browser (Book VI, Chapter 5) |
| **Screen-specific data shaping** | **API endpoint designed for the screen**, or BFF | Fewer round trips; client stays simple |

### Duplicated rules: when and how

Some rules must exist in more than one place. The question is how to keep copies
consistent:

- **Validation rules (client + server)**: generate the client's constraints from the API's
  OpenAPI document (Chapter 2), or keep a small, deliberate duplicate with tests on both sides.
- **Enumerations** (statuses, priorities): generate TypeScript types from the API contract.
- **Business rules needed in several layers** (SLA targets used by C# code, SQL reports and
  the UI): move them into **data** (an `sla_targets` table read by all three) or expose them
  through the API (`GET /api/config/sla`) so the client doesn't re-implement them.

> **⚠️ What can go wrong:** The most expensive duplication is **business logic in the
> frontend** that the API doesn't enforce, such as "agents can't reassign urgent tickets"
> implemented only by hiding a button. It works until someone calls the API directly, or a
> mobile app is built without the rule.

---

## 4. Designing APIs for screens

Book III designed a resource-oriented API: tickets, comments, users. A real screen,
Beacon's ticket detail page, needs the ticket, its comments, the reporter's and assignee's
names, the team, applicable SLA status, related knowledge-base articles, and the current
user's permissions on it. With strictly resource-oriented endpoints, that's six or seven
requests.

Options:

| Approach | How | Trade-off |
|---|---|---|
| **Many fine-grained calls** | The client fetches each resource | Simple API; many round trips; client-side joins |
| **Expanded representations** | `GET /tickets/T-42?include=comments,assignee` | Flexible; more complex endpoint; risk of over-fetching |
| **Screen-specific endpoints** | `GET /tickets/T-42/detail-view` | One round trip, exactly the data needed; couples API to UI |
| **BFF aggregation** | The BFF calls several APIs and composes a response | Keeps the core API clean; one round trip from the browser; BFF owns UI shaping |
| **GraphQL** | The client specifies exactly what it needs | Very flexible; significant complexity (Book III, Chapter 4) |

A balanced approach for Beacon:

- The **core API stays resource-oriented** for integrations and the mobile app.
- A few **view endpoints** exist for screens where round trips hurt (the ticket detail view
  returns the ticket with comments, people and permissions in one response).
- Permission flags come **from the server** (`"permissions": { "canResolve": true }`), so
  the UI reflects exactly the rules the API enforces, rather than re-implementing them.

```json
GET /api/tickets/T-42/view
{
  "ticket": { "id": "T-42", "title": "VPN drops", "status": "InProgress", "version": 7, ... },
  "comments": [ { "id": 101, "author": { "id": "u-7", "name": "Maria Lopez" }, "body": "...", "createdAt": "..." } ],
  "reporter": { "id": "u-2", "name": "Sam Lee" },
  "assignee": { "id": "u-7", "name": "Maria Lopez" },
  "sla": { "target": "PT4H", "breachesAt": "2026-10-07T13:12:00Z", "breaching": false },
  "permissions": { "canComment": true, "canResolve": true, "canReassign": false }
}
```

That `sla` object is the answer to the SLA duplication problem: the server computes it, and
the client just displays it.

---

## 5. Consistency across the stack

### Errors that survive the trip

An error should keep its meaning from the database to the user:

```text
 PostgreSQL unique violation (23505)
   → EF Core DbUpdateException
   → Infrastructure maps to Error.Conflict("A ticket with this title is already open.")
   → API returns 409 problem details { title, detail, traceId }
   → API client throws ApiError(409, problem)
   → Form shows the detail at form level; "Copy error ID" offers the traceId
```

Every layer translates the error into its own vocabulary **without losing information that
the next layer needs**. The `traceId` lets support jump from the user's screenshot to the
exact log lines (Book III, Chapter 10).

### Time

Store and transmit instants in **UTC with offsets** (`timestamptz` → `DateTimeOffset` →
ISO 8601 string → JavaScript `Date`/`Temporal`), and convert to the user's time zone only
for display, in the browser, using `Intl.DateTimeFormat`. "Business days" or "office hours"
rules (SLA pauses overnight) need an explicit time zone per team, stored as data.

### Identity

The same user is a `sub` claim in the token, a `users.id` row in the database, a
`ClaimsPrincipal` in the API and a `session.user.id` in React. Use one stable identifier
everywhere (the identity provider's `sub`), never email addresses (which change).

### Concurrency

Optimistic concurrency spans the whole stack: the database `version` column (Book IV), the
API's `ETag`/`If-Match` (Book III), and the client sending the version it loaded and handling
`412` by refreshing and showing "This ticket was changed by someone else."

---

## 6. Deploying a full-stack system

Layers are deployed separately, so for some minutes, **old and new versions run together**:
a new frontend talks to an old API, or an old frontend (in a user's open tab, for hours)
talks to a new API.

Rules that keep this safe:

1. **The API stays backward compatible** with the previous frontend version: additive
   changes only (Book III, Chapter 4).
2. **Deploy backend before frontend** for new features: the API gains the new field or
   endpoint first; the frontend starts using it afterwards.
3. **Database migrations are expand-and-contract** (Book IV, Chapter 7).
4. **Feature flags** decouple deployment from release: code ships dark, then turns on.
5. **Long-lived tabs**: an SPA loaded yesterday may still be running. Detect new versions
   (poll a `version.json`, or use a header) and prompt users to refresh; handle `404` for
   lazy-loaded chunks that no longer exist after a deploy by reloading.

---

## 7. In practice: Beacon's architecture, on one page

```text
                    ┌───────────────────────── Browser ─────────────────────────┐
                    │ React SPA: features/tickets, kb · TanStack Query cache    │
                    │ React Router (URL state) · SignalR client                 │
                    └──────────────┬───────────────────────────────▲────────────┘
                                   │ HTTPS, cookie, X-CSRF         │ WebSocket (via BFF)
                    ┌──────────────▼───────────────────────────────┴────────────┐
                    │ Beacon.Bff: OIDC login · cookie session · token storage   │
                    │ YARP proxy /api, /hubs · serves SPA static files          │
                    └──────────────┬────────────────────────────────────────────┘
                                   │ Bearer access token
   ┌──── Identity provider ◄───────┤
   │   (Entra ID / Keycloak)       │
   │                    ┌──────────▼────────────────────────────────────────────┐
   │                    │ Beacon.Api: minimal APIs · auth policies · validation │
   └── JWKS (keys) ────►│ TicketService (Core) · EF Core (Infrastructure)        │
                        │ SignalR hub · outbox worker · SLA monitor              │
                        └──────────┬──────────────────────────────┬─────────────┘
                                   │ SQL (TLS, beacon_app)        │ SMTP / webhooks
                        ┌──────────▼─────────┐            ┌───────▼────────┐
                        │ PostgreSQL 18      │            │ Email provider │
                        │ tickets, outbox... │            └────────────────┘
                        └────────────────────┘
```

Decisions this diagram records:

- **One public origin** (the BFF), so no CORS, and cookies are first-party.
- **The API is never called directly by browsers**, but is designed as a standalone
  resource server for other clients.
- **Business rules live in `Beacon.Core`**; the database enforces integrity; the UI renders
  server-computed permissions and SLA state.
- **Side effects go through the outbox**, so notifications and live updates are consistent
  with committed data.

### A full-stack checklist for new features

When Beacon adds a feature (say, ticket attachments), walk the stack:

1. **Data**: schema, constraints, indexes, migration plan (expand/contract?).
2. **Domain**: invariants and events (max attachment count; closed tickets reject uploads).
3. **API**: contract designed first; status codes; authorization per resource; limits
   (file size); backward compatibility.
4. **Storage and security**: where files live (blob storage, Book IX), malware scanning,
   content types, download authorization.
5. **Frontend**: URL state, query keys and invalidation, loading/error/empty states,
   accessibility, permission flags from the server.
6. **Real-time and side effects**: events, outbox, notifications.
7. **Observability**: logs, metrics, traces for the new paths.
8. **Tests**: domain unit tests, API integration tests, component tests with MSW, one E2E
   journey.
9. **Rollout**: feature flag, deployment order, monitoring after release.

That checklist is, in effect, a table of contents for this book.

---

## 8. What can go wrong

- **Chatty screens** with many sequential requests.
- **Business rules only in the frontend.**
- **Rules duplicated in several layers** with no mechanism keeping them in sync.
- **Errors losing meaning** across layers ("Something went wrong").
- **Time zone bugs** from converting at the wrong layer.
- **Breaking API changes** deployed while old frontends are still running.
- **Every layer reinventing the same concern** (three different retry policies, three logging formats).

---

## 9. How an experienced engineer thinks about this

- **Trace the request.** Knowing every hop is the basis for debugging, performance and
  security reasoning.
- **Place logic by authority**: security and invariants on the server, integrity in the
  database, presentation in the client.
- **Minimize round trips from the browser.**
- **Design the seams**: contracts, errors, time, identity and concurrency deserve explicit
  decisions.
- **Assume mixed versions** during every deployment.

---

## 10. Check yourself

**Questions**

1. Trace "add a comment" through Beacon. Where can it be rejected, and why does each check
   exist?
2. Why do browser round trips matter more than server processing time?
3. Where should authorization logic live? Business invariants? Presentation logic?
4. How can you keep validation rules consistent between client and server?
5. What are the options for serving a screen that needs data from many resources?
6. Why should permission flags come from the server?
7. What rules keep deployments safe when frontend and backend versions are mixed?

**Exercises**

1. Implement `GET /api/tickets/{id}/view` with comments, people, SLA and permissions, and
   switch the ticket detail page to it. Measure the difference in round trips.
2. Draw the sequence diagram for "resolve a ticket," including the outbox and live update.
3. Find a rule duplicated across layers in a system you know and propose how to give it a
   single source of truth.
4. Implement new-version detection in the SPA: poll `/version.json` and show a "New version
   available" banner.

**Interview-style questions**

- "Walk me through what happens when a user submits a form in your application."
- "Where would you put validation logic in a full-stack app?"
- "How do you deploy frontend and backend changes without breaking users?"

---

## 11. Going deeper

- Sam Newman, *Building Microservices* — the BFF pattern and API composition.
- [Azure Architecture Center: Backends for Frontends pattern](https://learn.microsoft.com/azure/architecture/patterns/backends-for-frontends)
- Martin Fowler, [Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)

**Next:** [Chapter 2 — Contracts Between Frontend and Backend](02-contracts-between-frontend-and-backend.md)
makes the API contract a single source of truth for both sides.
