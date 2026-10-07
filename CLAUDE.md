# Writing guide for this book

This repo is an mdBook: *The Modern Software Engineer*. `src/front/master-plan.md` is the
authoritative spec; read it before writing. Chapter 1 of Book I is the reference for voice
and depth.

## Conventions

- One file per chapter under `src/NN-book/`. Replace the "Status: planned" placeholder
  completely; keep the H1 title identical to `SUMMARY.md`.
- Follow the chapter template: problem → mental model → in practice (Beacon) → what can go
  wrong → when not to use it → how an experienced engineer thinks → check yourself
  (questions, exercises, interview-style questions) → going deeper → "Next:" link.
- Callouts: `> **🧱 Durable:**`, `> **🔄 Current (as of <Month Year>):**`,
  `> **⚠️ What can go wrong:**`, `> **🧭 When not to use it:**`, `> **🔍 Investigation:**`.
  Version-specific facts go only in 🔄 callouts.
- Target 4,000–7,000 words. Plain, direct prose; explain *why*; compare with C#/.NET when
  teaching other languages.
- Current baseline (October 2026): .NET 10 / C# 14 (LTS), PostgreSQL 18, TypeScript 7.0 (native Go compiler, GA July 2026; 6.0 was the last JS-based),
  React 19, Node 24 LTS, Python 3.14.

## Running project: Beacon

A team knowledge base + support desk. Solution `Beacon.sln` with `src/Beacon.Core`
(domain), `src/Beacon.Cli`, later `src/Beacon.Api` (ASP.NET Core), `src/Beacon.Infrastructure`
(EF Core/PostgreSQL), `tests/Beacon.Core.Tests` (xUnit), `web/` (React + TS + Vite).
State after Book I (keep consistent):
- `Beacon.Core.Tickets`: `TicketId` (readonly record struct, ToString "T-{n}"), `TicketStatus`,
  `TicketPriority {Low, Normal, High, Urgent}`, `Comment` record(Author, Body, CreatedAt),
  `Ticket` sealed class : `IEntity<TicketId>` — ctor(TicketId, title, priority, createdAt),
  AssignTo(assignee, now), AddComment(comment), Resolve(now), Close(), Reopen(), DomainEvents,
  ClearDomainEvents(); throws `DomainException` when closed. Events `TicketAssigned`, `TicketResolved`.
  `SlaRules` (ResponseTarget, IsBreaching), `TriageQueue`, `TicketSummary` record,
  `ITicketRepository` (FindAsync, SaveAsync), `TicketService(ITicketRepository, TimeProvider)`
  with `AddCommentAsync` returning `Result<Ticket>`.
- `Beacon.Core.Common`: `Result<T>` (readonly struct, Match), `Error(Code, Message)` with
  NotFound/Conflict/Validation (codes not_found/conflict/validation), `IEntity<TId>`,
  `IDomainEvent`, `EventDispatcher`, `DomainException`, `SensitiveAttribute`.
- Others: `Beacon.Core.Notifications.INotifier`, `Beacon.Core.Auditing.AuditLog`/`AuditFormatter`,
  `Beacon.Core.Indexing.IndexingQueue` (Channel), `Beacon.Core.Reports.AgentWorkloadReport`,
  `AddBeaconCore()` DI extension. CLI uses Generic Host + `CliApp`.
- Book II added `Ticket.Description`, `Ticket.Escalate()`. Book III added `Ticket.Version` (int, bumped on change),
  `ReporterId` (sub), `Team`; `TicketId : IParsable` (accepts "T-7" or "7"), `TicketIdJsonConverter` ("T-7"),
  `ICurrentUser`. Beacon.Api: minimal APIs in `Tickets/TicketEndpoints.cs` (MapGroup /api/tickets), contracts
  `CreateTicketRequest`, `AddCommentRequest`, `TicketResponse`, `Page<T>`/`TicketCursor` cursor paging,
  ETag/If-Match, Problem Details, JWT bearer + policies (roles customer/agent/lead; `TicketAuthorizationHandler`,
  `TicketOperations.Read/Work`), HybridCache, output cache for KB articles, `NotificationQueue`+`NotificationWorker`,
  SignalR `TicketHub` (/hubs/tickets), `SlaMonitor`, rate limiting, health checks. Data still in-memory
  (`InMemoryTicketRepository`) until Book IV.
- Book IV: PostgreSQL schema (tickets, comments, users(id=sub), teams, tags, ticket_tags, articles with
  tsvector `search`, outbox, sla_targets); `Ticket` now has `TeamId`, `AssigneeId`, `ReporterId`, `ResolvedAt`,
  `CustomFields` (jsonb). `src/Beacon.Infrastructure` with `BeaconDbContext` (snake_case), `EfTicketRepository`,
  `DomainEventsToOutboxInterceptor`, outbox worker (skip locked). DB roles beacon_migrator/app/readonly.
  Tests use Testcontainers (`BeaconApiFactory`).
- Books V–VI: `web/` (Vite + React 19 + TS strict, pnpm). `web/src/api/client.ts` (`ticketsApi`, `ApiError`,
  `X-CSRF: 1` header, 401 → `/bff/login`), `types.ts` (TICKET_STATUSES/PRIORITIES as const), branded `TicketId`,
  TanStack Query (`ticketKeys`, `ticketListQuery` infinite/cursor, `ticketDetailQuery`), React Router v7 lazy routes,
  React Hook Form + Zod (`NewTicketForm`, `toFieldErrors`), SignalR `useTicketUpdates`, `SessionContext`/`useCan`,
  Zustand `useUiStore` (density, liveStatus), feature folders `features/tickets`, `shared/ui`, Vitest + RTL + MSW.
  `src/Beacon.Bff` (cookie `__Host-beacon`, OIDC code flow, YARP proxy /api and /hubs to Beacon.Api, SPA fallback).
- Tests: `tests/Beacon.Api.Tests` (WebApplicationFactory<Program>), `tests/Beacon.Core.Tests` (xUnit, `TicketBuilder`, `FakeTicketRepository`, FakeTimeProvider).
- Books VII–IX: generated OpenAPI contract → TS types, Playwright E2E; hardened containers, Compose, Nginx;
  Azure Container Apps (`ca-beacon-bff` external via Front Door, `ca-beacon-api` internal, `ca-beacon-worker`,
  jobs `beacon-migrate`, `beacon-retention`), PostgreSQL Flexible Server, Azure Managed Redis, Blob Storage,
  Key Vault, App Configuration flags, Application Insights + SLOs, GitHub Actions with OIDC, Bicep, canary releases.
- Books X–XII: `beacon-tools` (Python, uv: import/export/evals); AI features (triage, summaries, reply drafts,
  RAG help assistant, incident agent) via Microsoft.Extensions.AI / Agent Framework / Foundry, pgvector;
  Rust `beacon-logscan` (Rayon) and `beacon-relay` (Tokio/axum webhook delivery, HMAC, SSRF guard).
- Book XIII: modular monolith (modules Tickets, Knowledge, Directory, Notifications, Assistant, Billing; per-module
  schemas, `*.Contracts` projects, architecture tests), integration events via outbox → Service Bus topic
  `beacon-events` (subscriptions notifications/search/webhooks/analytics), inbox tables, Idempotency-Key filter,
  multi-tenant (organizations, EF query filters + RLS via `beacon.tenant_id`), ADRs. Book XIV: Aspire AppHost,
  production readiness review.

## Publishing

After each chapter: update the book's `README.md` status line, add a line to
`src/front/changelog.md`, run `mdbook build`, commit, push to `main` (CI deploys to
GitHub Pages). Updates to *existing* content after review go through pull requests.

## Maintenance (monthly update run)

All 95 chapters are written. Ongoing work is keeping the book current:

1. Check every `🔄 Current (as of …)` callout (`grep -rn "🔄 Current" src`) and the baseline versions above
   against official sources (release notes, support policies, docs). Verify with web search before changing.
2. Fix factual drift only: versions, renamed products, changed APIs, end-of-support dates, new GA releases
   that change a recommendation. Don't rewrite durable (🧱) content or restyle chapters.
3. Update the callout's "as of" month for every callout you re-verify or change, and update the baseline here.
4. Work on a branch `update/YYYY-MM`, add one changelog entry per change under a dated header in
   `src/front/changelog.md`, run `mdbook build`, push the branch, and open a PR to `main` with the REST API
   (`gh api repos/awccon/engineering-book/pulls -f title=… -f head=update/YYYY-MM -f base=main -f body=…`;
   GraphQL is unavailable). Never push directly to `main` for updates; the owner reviews and merges.
5. The PR body lists each change with the source URL that justifies it. If nothing needs changing, open no PR
   and say so in the run summary.
