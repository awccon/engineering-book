# Contracts Between Frontend and Backend

Beacon's TypeScript types (`TicketSummary`, `CreateTicketInput`) were written by hand in
Book V, to match C# records written by hand in Book III. Today they agree. Tomorrow someone
renames `commentCount` to `commentsCount` in C#, the API ships, and the frontend shows
`undefined` everywhere, with no compiler error on either side.

This chapter makes the API contract **a single source of truth**: generated from the
backend, consumed by the frontend, checked in CI, and evolved without breaking clients.

---

## 1. The problem: two type systems, one wire format

The C# compiler checks the backend; the TypeScript compiler checks the frontend; nothing
checks the JSON in between. Hand-maintained types on both sides are a contract kept by
discipline, and discipline fails under deadlines.

Typical drift:

- Renamed or removed fields.
- Changed nullability (`assignee` was `string`, becomes `string | null`).
- New enum values the client doesn't handle.
- Different date formats, number precision, casing.
- Error responses that don't match what the client parses.

---

## 2. The mental model: contract-first or code-first, but one source

```text
 Code-first:      C# endpoints + types ──generate──► OpenAPI document ──generate──► TypeScript types / client
 Contract-first:  OpenAPI document (hand-written) ──generate──► C# server stubs + TypeScript client
```

Either way, there is **one** authoritative description of the API, and both sides are
generated from it or verified against it.

| | Code-first | Contract-first |
|---|---|---|
| Source of truth | Backend code | OpenAPI (YAML/JSON) document |
| Best when | One team owns both sides; backend evolves quickly | Many teams or external consumers; API designed before implementation |
| Risk | API shape is an accident of implementation | Generated server code can be awkward; document and code can drift without checks |

Beacon is **code-first**: ASP.NET Core generates the OpenAPI document (Book III, Chapter 2),
and the frontend generates its types from it. Book III, Chapter 4's "design before code"
still happens, as reviewed request/response sketches, before the endpoints are written.

### OpenAPI in one paragraph

**OpenAPI** (formerly Swagger) is a standard, language-neutral description of an HTTP API:
paths, methods, parameters, request bodies, responses (per status code), schemas (types),
and security schemes. It powers documentation UIs, client generators, mock servers,
contract tests and API gateways.

---

## 3. Making the OpenAPI document accurate

A generated contract is only as good as the metadata the backend provides. In ASP.NET Core
minimal APIs:

- **Typed results** (`Results<Ok<TicketResponse>, NotFound, ValidationProblem>`) document
  every status code and its body automatically.
- **Nullable reference types** become `nullable` in the schema, so `string?` in C# becomes
  `string | null` in TypeScript.
- **`required` members** and non-nullable properties become required fields.
- **Enums**: with `JsonStringEnumConverter`, they appear as string enums (`"Open" |
  "InProgress" | ...`).
- **Summaries and descriptions** (`.WithSummary`, `.WithDescription`, XML comments) become
  documentation.
- **Custom types** need help: `TicketId` serializes as a string via a custom converter (Book
  III, Chapter 5), so the schema must say so. A **schema transformer** fixes it:

```csharp
builder.Services.AddOpenApi(o =>
{
    o.AddSchemaTransformer((schema, context, ct) =>
    {
        if (context.JsonTypeInfo.Type == typeof(TicketId))
        {
            schema.Type = JsonSchemaType.String;
            schema.Pattern = "^T-\\d+$";
            schema.Properties?.Clear();
        }
        return Task.CompletedTask;
    });
});
```

> **🔄 Current (as of October 2026):** ASP.NET Core 10's built-in OpenAPI generation produces
> OpenAPI 3.1 documents by default and supports document, operation and schema
> transformers. The exact transformer APIs evolved between .NET 9 and 10; check the docs for
> your version.

### Export the document at build time

Rather than fetching it from a running server, generate the file during the build so it can
be committed and diffed:

```xml
<!-- src/Beacon.Api/Beacon.Api.csproj -->
<PackageReference Include="Microsoft.Extensions.ApiDescription.Server" Version="10.*" PrivateAssets="all" />
<PropertyGroup>
  <OpenApiDocumentsDirectory>$(MSBuildProjectDirectory)/../../contracts</OpenApiDocumentsDirectory>
</PropertyGroup>
```

`dotnet build` now writes `contracts/Beacon.Api.json`. **Commit it.** Every API change then
shows up as a diff of the contract in the pull request, which reviewers can read (Book II,
Chapter 3): "this PR removes the `assignee` field from `TicketResponse`" is impossible to
miss.

---

## 4. Generating the client side

Several tools generate TypeScript from OpenAPI:

| Tool | Generates | Notes |
|---|---|---|
| **openapi-typescript** + **openapi-fetch** | Types only + a tiny typed `fetch` wrapper | Lightweight; types derived at compile time; no generated runtime code |
| **Orval**, **Hey API (openapi-ts)**, **Kiota** | Full clients, optionally TanStack Query hooks, Zod schemas, MSW mocks | More features; more generated code |
| **NSwag** | C# and TypeScript clients | Popular in .NET shops |

Beacon uses **openapi-typescript** for types plus its existing hand-written client functions,
because the client layer is small and the types are what matter:

```bash
cd web
pnpm add -D openapi-typescript
pnpm add openapi-fetch
```

```json
// web/package.json (scripts)
"generate:api": "openapi-typescript ../contracts/Beacon.Api.json -o src/shared/api/schema.d.ts"
```

The generated file contains `paths` and `components` types:

```ts
// excerpt of the generated schema.d.ts
export interface components {
  schemas: {
    TicketResponse: {
      id: string;
      title: string;
      description: string | null;
      status: 'Open' | 'InProgress' | 'Resolved' | 'Closed';
      priority: 'Low' | 'Normal' | 'High' | 'Urgent';
      assignee: string | null;
      createdAt: string;
      commentCount: number;
      version: number;
    };
    // ...
  };
}
```

The hand-written types from Book V are replaced by aliases into the generated ones:

```ts
// web/src/shared/api/types.ts
import type { components } from './schema';

export type TicketSummary = components['schemas']['TicketResponse'];
export type CreateTicketInput = components['schemas']['CreateTicketRequest'];
export type TicketStatus = TicketSummary['status'];
export type TicketPriority = TicketSummary['priority'];
export type ProblemDetails = components['schemas']['ProblemDetails'];
```

And `openapi-fetch` gives fully typed requests with zero hand-written path strings:

```ts
// web/src/shared/api/http.ts
import createClient from 'openapi-fetch';
import type { paths } from './schema';

export const api = createClient<paths>({
  baseUrl: '/',
  credentials: 'include',
  headers: { 'X-CSRF': '1' },
});

// usage: path, params and response are all typed from the contract
const { data, error, response } = await api.GET('/api/tickets/{id}', {
  params: { path: { id: ticketId } },
  signal,
});
```

Typos in paths, missing required parameters, and wrong body shapes are now compile errors.

### Enum values and exhaustive handling

The status union comes from the contract. When the backend adds `"Escalated"`, regenerating
types changes `TicketStatus`, and every exhaustive `switch` and `satisfies Record<TicketStatus, …>`
in the frontend (Book V, Chapter 5) fails to compile until it's handled. That's the contract
pipeline paying for itself.

---

## 5. Keeping the contract honest in CI

Generation only helps if it runs. Two CI checks close the loop:

**1. The committed contract matches the code.** Build the API, regenerate the document, and
fail if it differs from what's committed:

```yaml
- run: dotnet build src/Beacon.Api -c Release
- run: git diff --exit-code contracts/Beacon.Api.json
```

**2. The frontend compiles against the current contract.**

```yaml
- run: pnpm --dir web generate:api
- run: git diff --exit-code web/src/shared/api/schema.d.ts
- run: pnpm --dir web exec tsc -b --noEmit
```

Now a backend change that breaks the frontend fails the pull request that introduced it,
not production.

### Detecting breaking changes

Diffing the contract shows *what* changed; tools can classify *whether it breaks clients*.
**oasdiff** compares two OpenAPI documents and reports breaking changes (removed fields,
new required parameters, narrowed types, removed enum values):

```yaml
- run: oasdiff breaking origin/main:contracts/Beacon.Api.json contracts/Beacon.Api.json --fail-on ERR
```

Combined with Book III, Chapter 4's evolution rules, that turns "don't break clients" from a
convention into an automated gate. (A deliberate breaking change then requires a new
version, or an explicit, reviewed override.)

---

## 6. Contract testing between services

When the **consumer** of an API is a different team or service, you want to know before
deployment that the provider still satisfies what consumers actually use. **Consumer-driven
contract testing** (Pact is the best-known tool):

1. The consumer's tests record the interactions it relies on (request → expected response
   shape) into a **pact** file.
2. The provider's CI replays those interactions against the real provider and verifies the
   responses.
3. A broker tracks which versions are compatible, and deployment tools ask "can I deploy
   this version?"

> **🧭 When not to use consumer-driven contracts:** When one team owns both sides in one
> repository, a generated, committed OpenAPI contract with CI checks (sections 3–5) gives
> most of the safety with far less machinery. Pact earns its keep with many independent
> teams and services deployed separately (Book XIII).

---

## 7. Mocks from the contract

The contract can also drive test doubles:

- **MSW handlers** typed against `paths` (Book VI, Chapter 7), so a mock returning the wrong
  shape fails to compile.
- **Mock servers** (Prism, or generated MSW handlers from Orval) let frontend work start
  before the backend endpoint exists: contract-first development within a code-first
  workflow.

```ts
// web/src/test/handlers.ts (typed)
import { http, HttpResponse } from 'msw';
import type { components } from '../shared/api/schema';

type TicketResponse = components['schemas']['TicketResponse'];

http.get('/api/tickets/:id', ({ params }) =>
  HttpResponse.json<TicketResponse>(aTicket({ id: String(params.id) })));
```

---

## 8. In practice: Beacon's contract pipeline

Putting it together:

```text
 Developer changes TicketResponse in C#
        │
        ▼
 dotnet build ──► contracts/Beacon.Api.json updated (committed in the same PR)
        │
        ▼
 pnpm generate:api ──► web/src/shared/api/schema.d.ts updated (committed)
        │
        ▼
 tsc ──► frontend compile errors wherever the change matters (fixed in the same PR)
        │
        ▼
 CI: contract up to date? frontend types up to date? oasdiff: breaking? tsc passes?
        │
        ▼
 Reviewers see the contract diff and the frontend changes side by side
```

The pull request template (Book II, Chapter 3) gains a line:

```markdown
## API contract
- [ ] No contract change
- [ ] Additive change (new fields/endpoints)
- [ ] Breaking change: version bump / migration plan described below
```

A worked example: product asks for "first response time" on the ticket list.

1. Add `FirstResponseAt` (`DateTimeOffset?`) to `TicketResponse` in C#, computed in the list
   query projection (Book IV, Chapter 7).
2. Build: the contract diff shows a new nullable field `firstResponseAt: string | null`.
   `oasdiff` reports it as non-breaking.
3. Regenerate types: `TicketSummary` gains the field; nothing breaks.
4. Use it in `TicketRow`, formatting with `Intl.RelativeTimeFormat` in the browser.
5. One pull request, reviewed as a unit, with both sides consistent by construction.

---

## 9. What can go wrong

- **Hand-maintained types on both sides** drifting silently.
- **Inaccurate OpenAPI**: untyped `IResult` returns, missing status codes, custom types
  described wrongly. Generated clients are then confidently wrong.
- **Generated code not committed or not checked**, so it's stale.
- **Breaking changes merged unnoticed** because nobody reads JSON diffs: automate detection.
- **Over-generated clients** with large runtime code nobody understands.
- **Clients that can't handle new enum values**, making additive changes breaking in practice.

---

## 10. How an experienced engineer thinks about this

- **One source of truth for the contract**, generated or verified on both sides.
- **The contract is reviewed like code**: committed, diffed, gated.
- **Make the compiler find consumers of a change**, rather than people.
- **Automate breaking-change detection**; treat breaking changes as exceptional and explicit.
- **Choose the lightest tooling that closes the loop** for your team structure.

---

## 11. Check yourself

**Questions**

1. Compare code-first and contract-first API development.
2. What backend metadata makes a generated OpenAPI document accurate?
3. Why commit the generated OpenAPI document and TypeScript types?
4. What two CI checks keep the contract honest?
5. What does a tool like oasdiff add?
6. When is consumer-driven contract testing (Pact) worth it?
7. How does a new enum value propagate through Beacon's pipeline?

**Exercises**

1. Set up build-time OpenAPI generation for Beacon.Api and commit the document.
2. Generate TypeScript types with openapi-typescript and replace Book V's hand-written types.
3. Rename a response field in C# and watch the pipeline catch it: contract diff, oasdiff,
   frontend compile errors.
4. Type Beacon's MSW handlers against the generated `paths`.

**Interview-style questions**

- "How do you keep frontend and backend types in sync?"
- "How do you detect breaking API changes before they reach production?"
- "What's contract testing?"

---

## 12. Going deeper

- [OpenAPI Specification](https://spec.openapis.org/oas/latest.html)
- [Microsoft docs: Generate OpenAPI documents in ASP.NET Core](https://learn.microsoft.com/aspnet/core/fundamentals/openapi/aspnetcore-openapi)
- [openapi-typescript](https://openapi-ts.dev/) and [oasdiff](https://www.oasdiff.com/)
- [Pact documentation](https://docs.pact.io/)

**Next:** [Chapter 3 — End-to-End Testing and Environments](03-end-to-end-testing-and-environments.md)
verifies the whole system works together.
