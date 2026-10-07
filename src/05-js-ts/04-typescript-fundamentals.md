# TypeScript Fundamentals

> **🔄 Current (as of October 2026):** TypeScript 7.0 (released July 2026) is the first
> version built on the new **native compiler written in Go**, roughly 10× faster than the
> JavaScript-based compiler for type checking large projects. TypeScript 6.0 (March 2026)
> was the last JavaScript-based release and served as a bridge, deprecating legacy
> options. The language itself is unchanged by the port; everything in this chapter applies
> to both.

For a C# developer, TypeScript looks immediately familiar: classes, interfaces, generics,
`async`/`await`, enums, access modifiers. That familiarity is useful and misleading.
TypeScript's type system works on fundamentally different principles from C#'s: types are
**structural**, not nominal; they **disappear at run time**; and they describe the
existing, dynamic JavaScript world rather than define a new one.

This chapter covers TypeScript's core ideas with that contrast in mind, and types Beacon's
API client.

---

## 1. The problem: JavaScript at scale

JavaScript's flexibility becomes a liability in large codebases:

- Typos create new properties instead of errors.
- Nothing tells you what shape of object a function expects.
- Renaming a property means searching text and hoping.
- `undefined is not a function` appears in production, not in the editor.

TypeScript adds a **static type layer** that catches these mistakes before the code runs,
powers editor features (autocomplete, go-to-definition, safe rename), and documents
intent, while compiling to plain JavaScript that runs anywhere.

---

## 2. The mental model

### Types are erased

```ts
function priorityLabel(p: TicketPriority): string { /* ... */ }
```

compiles to:

```js
function priorityLabel(p) { /* ... */ }
```

The types are checked at compile time and **removed**. At run time there are no types, no
interfaces, no generic type information. Consequences:

- You **can't check an interface at run time** (`if (x instanceof TicketDto)` doesn't exist
  for interfaces).
- **Data from outside** (API responses, `JSON.parse`, `localStorage`, form input) has
  whatever shape it actually has, regardless of the type you annotate. TypeScript trusts
  your annotation. Validate at boundaries (section 7).
- TypeScript can't change JavaScript behavior: `0.1 + 0.2` is still `0.30000000000000004`.

### Structural typing

In C#, a type is compatible with another only if it **declares** the relationship
(`class Ticket : IEntity`). That's **nominal** typing. TypeScript uses **structural**
typing: a value is compatible with a type if it **has the right shape**.

```ts
interface HasId { id: string }

function logId(x: HasId) { console.log(x.id); }

const ticket = { id: 'T-1', title: 'VPN drops' };
logId(ticket);               // ✓ ticket has an id: string, so it's a HasId
```

`ticket` never mentions `HasId`. It just fits. This matches how JavaScript code actually
works (passing object literals around) and makes TypeScript flexible. It also means two
unrelated types with the same shape are interchangeable: a `UserId` string and a `TicketId`
string are both just `string` (Chapter 5 shows how to get nominal-style "branded" types).

### Types as sets of values

A helpful way to think about TypeScript types: **a type is a set of possible values.**

- `string` is the set of all strings; `'Open'` is a set with one value.
- `'Open' | 'Resolved'` is the union (both values).
- `{ id: string }` is the set of all objects that have an `id` string property (and possibly
  more).
- `never` is the empty set; `unknown` is the set of everything.

Assignability is subset-ness: you can assign `'Open'` to `string` because `{'Open'}` ⊂
strings.

---

## 3. Basic types

```ts
let title: string = 'VPN drops';
let count: number = 3;
let done: boolean = false;
let tags: string[] = ['network', 'vpn'];          // or Array<string>
let pair: [string, number] = ['T-1', 3];          // tuple
let anything: unknown = JSON.parse(text);          // must be narrowed before use
let escape: any = legacyLib();                     // turns off checking: avoid
```

### Type inference

You rarely need annotations on locals; TypeScript infers them:

```ts
const ids = tickets.map(t => t.id);   // inferred: string[]
let status = 'Open';                  // inferred: string
const fixed = 'Open';                 // inferred: 'Open' (a const can't change, so its type is the literal)
```

Annotate **function parameters**, **public return types** (documents intent and catches
mistakes inside the function), and places where inference picks something too wide.

### `any` vs `unknown`

- `any` disables type checking for that value *and everything it touches*. It spreads
  silently. Treat it as a bug.
- `unknown` means "could be anything; prove what it is before using it." It's the correct
  type for untrusted data.

```ts
function parse(text: string): unknown {
  return JSON.parse(text);
}
const data = parse(input);
data.title;                              // ✗ error: data is unknown
if (typeof data === 'object' && data !== null && 'title' in data) { /* narrowed */ }
```

---

## 4. Object types: interfaces and type aliases

```ts
interface TicketSummary {
  id: string;
  title: string;
  status: TicketStatus;
  priority: TicketPriority;
  assignee: string | null;
  readonly createdAt: string;      // can't be reassigned
  commentCount?: number;           // optional: may be missing
}

type TicketStatus = 'Open' | 'InProgress' | 'Resolved' | 'Closed';
type TicketPriority = 'Low' | 'Normal' | 'High' | 'Urgent';
```

### Interface or type alias?

Both describe object shapes. Differences:

| | `interface` | `type` |
|---|---|---|
| Object shapes | ✓ | ✓ |
| Unions, tuples, primitives, mapped types | ✗ | ✓ |
| Extending | `extends` | `&` (intersection) |
| Declaration merging (adding to an existing interface) | ✓ | ✗ |

A common convention: `type` everywhere unless you need declaration merging or prefer
`interface` for public object contracts. Consistency matters more than the choice.

### Optional vs nullable

```ts
assignee: string | null;     // always present; may be null
assignee?: string;           // may be missing entirely (undefined)
```

These are different, and the difference matters for API contracts (Book III, Chapter 5's
"absent vs null"). With `exactOptionalPropertyTypes` enabled, TypeScript also distinguishes
"missing" from "present but undefined."

### Excess property checks

Structural typing allows extra properties, *except* in one place: fresh object literals
assigned directly to a typed target are checked for extra properties, which catches typos:

```ts
const t: TicketSummary = { id: 'T-1', titel: 'x', /* ... */ };   // ✗ 'titel' does not exist
```

---

## 5. Union types and narrowing

**Union types** are the feature C# developers find most new and most powerful. A value can
be one of several types, and TypeScript tracks which one it is through **control flow
analysis**:

```ts
function describe(assignee: string | null): string {
  if (assignee === null) {
    return 'Unassigned';              // here: assignee is null
  }
  return assignee.toUpperCase();      // here: assignee is string
}
```

Narrowing works with `typeof`, `instanceof`, `in`, equality checks, truthiness, and
user-defined type guards:

```ts
function isApiError(e: unknown): e is ApiError {
  return e instanceof Error && e.name === 'ApiError';
}
```

### Discriminated unions

The pattern you'll use most: a union of object types sharing a **literal "tag" property**:

```ts
type LoadState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: Error };

function render(state: LoadState<TicketSummary[]>) {
  switch (state.status) {
    case 'idle':    return 'Nothing yet';
    case 'loading': return 'Loading…';
    case 'success': return `${state.data.length} tickets`;     // data available only here
    case 'error':   return `Failed: ${state.error.message}`;   // error available only here
  }
}
```

This makes **illegal states unrepresentable** (Book I, Chapter 2): there's no way to have
`status: 'loading'` with `data`, or `error` and `data` at the same time, which a type with
`isLoading: boolean; data?: T; error?: Error` would allow.

### Exhaustiveness checking

```ts
function assertNever(x: never): never {
  throw new Error(`Unexpected value: ${JSON.stringify(x)}`);
}

// add to the switch above:
    default: return assertNever(state);
```

If someone adds `{ status: 'refreshing' }` to the union and forgets this switch, the
compiler errors: `state` is no longer `never` in the default branch. This is the closed
"set of cases" approach from Book I, Chapter 3, with compiler help.

---

## 6. Functions, classes and enums

### Functions

```ts
function createTicket(title: string, priority: TicketPriority = 'Normal', tags?: string[]): Ticket { /* ... */ }

type Comparer<T> = (a: T, b: T) => number;       // function type

function on(event: 'created', handler: (t: Ticket) => void): void;   // overloads
function on(event: 'deleted', handler: (id: string) => void): void;
function on(event: string, handler: (arg: any) => void): void { /* implementation */ }
```

### Classes

TypeScript classes look like C#'s, with some differences:

```ts
class TicketStore {
  readonly #items = new Map<string, TicketSummary>();     // JS private field (runtime-private)
  private cacheHits = 0;                                    // TS private (compile-time only!)

  constructor(private readonly api: TicketsApi) {}          // parameter property: declares and assigns

  get count(): number { return this.#items.size; }
}
```

`private` is checked only by the compiler; at run time the property is visible. `#private`
is enforced by JavaScript itself. And remember: classes satisfy interfaces structurally;
`implements` is optional documentation that the compiler verifies.

### Enums: prefer unions

TypeScript `enum`s are one of its few features that generate run-time code, and they have
quirks (numeric enums accept any number; reverse mappings). For most uses, a **union of
string literals** is simpler and serializes naturally to and from JSON:

```ts
type TicketStatus = 'Open' | 'InProgress' | 'Resolved' | 'Closed';

// When you also need the list of values at run time:
export const TICKET_STATUSES = ['Open', 'InProgress', 'Resolved', 'Closed'] as const;
export type TicketStatus = (typeof TICKET_STATUSES)[number];
```

`as const` makes the array readonly and its elements literal types; the type is derived
from the value, so they can't drift apart.

---

## 7. Validating data at the boundary

Since types are erased, **TypeScript cannot verify that an API response has the shape you
declared**. This is the same lesson as Book III, Chapter 5 ("parse, don't validate"), from
the other side of the wire:

```ts
const ticket = (await response.json()) as TicketSummary;   // a promise to the compiler, not a check
```

If the API changes, renames a field or returns an error body, the code compiles and fails
somewhere far from the cause. Options:

1. **Trust a generated client** whose types come from the API's OpenAPI document (Book VII,
   Chapter 2). Mismatches are caught when you regenerate.
2. **Validate at run time** with a schema library such as **Zod** or **Valibot**, which
   gives you both a validator and the TypeScript type from one definition:

```ts
import { z } from 'zod';

export const TicketSummarySchema = z.object({
  id: z.string().regex(/^T-\d+$/),
  title: z.string(),
  status: z.enum(['Open', 'InProgress', 'Resolved', 'Closed']),
  priority: z.enum(['Low', 'Normal', 'High', 'Urgent']),
  assignee: z.string().nullable(),
  createdAt: z.string().datetime({ offset: true }),
  commentCount: z.number().int().nonnegative(),
});

export type TicketSummary = z.infer<typeof TicketSummarySchema>;   // type derived from the schema

const ticket = TicketSummarySchema.parse(await response.json());   // throws with details if wrong
```

Validating every response costs some CPU and bundle size; many teams validate at the
boundaries most likely to drift (third-party APIs, `localStorage`, URL parameters) and rely
on generated types for their own API.

---

## 8. Strict mode and the compiler options that matter

`"strict": true` enables a family of checks you want on every project. The most important:

| Option | What it catches |
|---|---|
| `strictNullChecks` | `null` and `undefined` aren't assignable to other types (like C#'s nullable reference types, but enforced) |
| `noImplicitAny` | Parameters and variables without inferable types must be annotated |
| `strictFunctionTypes` | Correct variance for function parameters |
| `useUnknownInCatchVariables` | `catch (e)` gives `unknown`, not `any` |

Beyond `strict`, Beacon enables (Chapter 3's tsconfig):

- **`noUncheckedIndexedAccess`**: `arr[0]` and `record[key]` have type `T | undefined`,
  because the index may not exist. Annoying at first; catches real bugs.
- **`exactOptionalPropertyTypes`**: `prop?: string` means "missing or string," not "or
  explicitly undefined."
- **`verbatimModuleSyntax`**: type-only imports must use `import type`, so the transpiler
  knows what to erase.

> **🧭 When not to loosen it:** It's tempting to turn off strict checks or sprinkle `any`
> to make errors go away during a migration or a deadline. Each one is a hole the compiler
> can no longer see through. Use `// @ts-expect-error` with a comment for genuine exceptions,
> so the suppression fails loudly once it's no longer needed.

---

## 9. In practice: typing Beacon's API client

Convert Chapter 2's `api.js` to TypeScript.

```ts
// web/src/api/types.ts
export const TICKET_STATUSES = ['Open', 'InProgress', 'Resolved', 'Closed'] as const;
export type TicketStatus = (typeof TICKET_STATUSES)[number];

export const TICKET_PRIORITIES = ['Low', 'Normal', 'High', 'Urgent'] as const;
export type TicketPriority = (typeof TICKET_PRIORITIES)[number];

export interface TicketSummary {
  id: string;
  title: string;
  status: TicketStatus;
  priority: TicketPriority;
  assignee: string | null;
  createdAt: string;            // ISO 8601; parse with care (Date or Temporal) for display
  commentCount: number;
  version: number;
}

export interface Page<T> {
  items: T[];
  nextCursor: string | null;
}

export interface CreateTicketInput {
  title: string;
  priority: TicketPriority;
  description?: string;
}

export interface ProblemDetails {
  type?: string;
  title?: string;
  status?: number;
  detail?: string;
  traceId?: string;
  errors?: Record<string, string[]>;
}
```

```ts
// web/src/api/client.ts
import type { CreateTicketInput, Page, ProblemDetails, TicketSummary, TicketStatus } from './types';

export class ApiError extends Error {
  override readonly name = 'ApiError';
  constructor(readonly status: number, readonly problem?: ProblemDetails) {
    super(problem?.detail ?? problem?.title ?? `HTTP ${status}`);
  }

  fieldErrors(): Record<string, string[]> {
    return this.problem?.errors ?? {};
  }
}

type HttpMethod = 'GET' | 'POST' | 'PATCH' | 'DELETE';

interface RequestOptions {
  method?: HttpMethod;
  body?: unknown;
  signal?: AbortSignal;
  headers?: Record<string, string>;
}

async function request<T>(path: string, options: RequestOptions = {}): Promise<T> {
  const { method = 'GET', body, signal, headers = {} } = options;

  const response = await fetch(`/api${path}`, {
    method,
    signal,
    credentials: 'include',
    headers: {
      Accept: 'application/json',
      ...(body !== undefined ? { 'Content-Type': 'application/json' } : {}),
      ...headers,
    },
    body: body === undefined ? undefined : JSON.stringify(body),
  });

  if (response.status === 204) return undefined as T;

  const isJson = response.headers.get('Content-Type')?.includes('json') ?? false;
  const payload: unknown = isJson ? await response.json() : await response.text();

  if (!response.ok) {
    throw new ApiError(response.status, isJson ? (payload as ProblemDetails) : undefined);
  }
  return payload as T;     // trusted: our own API, with generated types in Book VII
}

export interface ListTicketsParams {
  status?: TicketStatus;
  assignee?: string;
  limit?: number;
  cursor?: string;
}

function toQuery(params: Record<string, string | number | undefined>): string {
  const entries = Object.entries(params).filter(
    (e): e is [string, string | number] => e[1] !== undefined,
  );
  return new URLSearchParams(entries.map(([k, v]) => [k, String(v)])).toString();
}

export const ticketsApi = {
  list: (params: ListTicketsParams = {}, signal?: AbortSignal) =>
    request<Page<TicketSummary>>(`/tickets?${toQuery({ ...params })}`, { signal }),

  get: (id: string, signal?: AbortSignal) =>
    request<TicketSummary>(`/tickets/${encodeURIComponent(id)}`, { signal }),

  create: (input: CreateTicketInput) =>
    request<TicketSummary>('/tickets', {
      method: 'POST',
      body: input,
      headers: { 'Idempotency-Key': crypto.randomUUID() },
    }),

  addComment: (id: string, body: string) =>
    request<TicketSummary>(`/tickets/${encodeURIComponent(id)}/comments`, { method: 'POST', body: { body } }),
} as const;
```

What the types buy us immediately:

- `ticketsApi.list({ status: 'Opne' })` is a compile error (typo in a literal union).
- `ticketsApi.create({ title: 'x' })` is an error: `priority` is required.
- Callers get `Page<TicketSummary>`, so `page.items[0].assignee` is known to be
  `string | null`, and with `noUncheckedIndexedAccess`, `page.items[0]` is
  `TicketSummary | undefined` until checked.
- The type-guard predicate in `toQuery` (`(e): e is [string, string | number]`) narrows the
  filtered entries, so no `undefined` reaches `String(v)`.

Note the two `as` casts in `request`. They're the honest boundary: this is where untrusted
JSON becomes a typed value. Book VII replaces the hand-written types with ones generated
from Beacon.Api's OpenAPI document, so the server and client can't silently disagree.

---

## 10. What can go wrong

- **`any` spreading** and silently disabling checks.
- **Type assertions (`as`) treated as validation.** They're claims, not checks.
- **Trusting API responses** without validation or generated types.
- **Confusing optional and nullable.**
- **Numeric enums** accepting any number; prefer string literal unions.
- **`private` assumed to be private at run time.**
- **Loose compiler options** letting nulls and missing indexes slip through.
- **Over-engineering types** until they're harder to read than the code (Chapter 5).

---

## 11. How an experienced engineer thinks about this

- **Types describe JavaScript; they don't change it.** Keep run-time behavior in mind.
- **Structural typing is a feature.** Design types around shapes and let values fit them.
- **Model states with discriminated unions.** Make illegal states unrepresentable.
- **Validate at the boundaries; trust inside.** Same principle as the backend.
- **Strict by default.** Loosening is a deliberate, documented exception.

---

## 12. Check yourself

**Questions**

1. What does it mean that TypeScript types are erased? Name two consequences.
2. Explain structural vs nominal typing with an example.
3. What's the difference between `any` and `unknown`?
4. What's a discriminated union, and how does exhaustiveness checking work?
5. Why prefer string literal unions over `enum`s?
6. Why is `response.json() as T` not validation? What are the alternatives?
7. What does `noUncheckedIndexedAccess` change?

**Exercises**

1. Write a `LoadState<T>` union and a `render` function with exhaustiveness checking; add a
   new state and watch the compiler find every switch to update.
2. Define a Zod schema for `Page<TicketSummary>` and validate a real response from Beacon.Api.
3. Turn on `noUncheckedIndexedAccess` in a project and fix the resulting errors. Which were
   real bugs?
4. Write a type guard `isTicketStatus(x: string): x is TicketStatus` using `TICKET_STATUSES`.

**Interview-style questions**

- "How does TypeScript's type system differ from C#'s or Java's?"
- "What's the difference between `interface` and `type`?"
- "How do you handle untrusted data in TypeScript?"
- "What's a discriminated union, and why is it useful?"

---

## 13. Going deeper

- [The TypeScript Handbook](https://www.typescriptlang.org/docs/handbook/intro.html)
- [TypeScript for C# programmers](https://www.typescriptlang.org/docs/handbook/typescript-in-5-minutes-oop.html)
- Dan Vanderkam, *Effective TypeScript*.
- [Zod documentation](https://zod.dev/)

**Next:** [Chapter 5 — TypeScript in Depth](05-typescript-in-depth.md) covers generics,
type-level programming and designing types that help rather than hinder.
