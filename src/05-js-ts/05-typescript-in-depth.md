# TypeScript in Depth

TypeScript's type system is far more expressive than C#'s. Beyond generics, it can compute
new types from existing ones: "the same object, but every property optional," "the keys of
this object whose values are functions," "the return type of this async function, unwrapped."
This is called **type-level programming**, and it's what lets libraries like React, Zod and
TanStack Query offer APIs that feel magical: you write plain code, and the types follow.

The same power can produce types nobody can read. This chapter covers the tools, how they
work, and the judgment to know when a clever type helps and when it hurts.

---

## 1. The problem: keeping types in sync without repeating yourself

Without type-level tools, related types drift apart:

```ts
interface Ticket { id: string; title: string; status: TicketStatus; priority: TicketPriority; /* ... */ }
interface TicketUpdate { title?: string; status?: TicketStatus; priority?: TicketPriority; }   // copy-paste
interface TicketFormValues { title: string; priority: TicketPriority; }                      // copy-paste
```

Add a field to `Ticket`, forget `TicketUpdate`, and nothing complains. TypeScript's answer:
**derive** types from a single source of truth.

```ts
type TicketUpdate = Partial<Pick<Ticket, 'title' | 'status' | 'priority'>>;
type TicketFormValues = Pick<Ticket, 'title' | 'priority'>;
```

---

## 2. The mental model: types as functions over types

Think of generic types as **functions that take types and return types**:

```ts
type Box<T> = { value: T };          // Box is a "function": Box(string) = { value: string }
type Nullable<T> = T | null;
```

TypeScript provides operators to build such functions:

| Operator | Meaning | Analogy |
|---|---|---|
| `keyof T` | Union of `T`'s property names | "list the fields" |
| `T[K]` | Type of property `K` of `T` (indexed access) | "get field type" |
| `typeof value` | The type of a value | "type of this variable" |
| `{ [K in Keys]: ... }` | Mapped type: build an object type by iterating keys | `foreach` over keys |
| `A extends B ? X : Y` | Conditional type | `if` |
| `infer U` | Capture part of a type inside a conditional | pattern-matching variable |
| Template literal types `` `on${Capitalize<K>}` `` | Build string literal types | string interpolation |

---

## 3. Generics

### Generic functions and constraints

```ts
function first<T>(items: readonly T[]): T | undefined {
  return items[0];
}

function byId<T extends { id: string }>(items: readonly T[]): Map<string, T> {
  return new Map(items.map(i => [i.id, i]));
}
```

Constraints (`extends`) work like C#'s `where T : ...`, but structurally: any type with an
`id: string` qualifies.

### Generic inference

TypeScript infers type arguments from usage, often more aggressively than C#:

```ts
const m = byId(tickets);      // T inferred as TicketSummary; m: Map<string, TicketSummary>
```

### `keyof` with generics: type-safe property access

```ts
function pluck<T, K extends keyof T>(items: readonly T[], key: K): T[K][] {
  return items.map(i => i[key]);
}

pluck(tickets, 'priority');    // TicketPriority[]
pluck(tickets, 'prority');     // ✗ error: not a key of TicketSummary
```

The return type `T[K][]` changes with the argument: something C#'s type system can't
express.

### `const` type parameters and `satisfies`

```ts
function defineRoutes<const T extends Record<string, string>>(routes: T): T { return routes; }
const routes = defineRoutes({ tickets: '/tickets', ticket: '/tickets/:id' });
// routes.ticket: '/tickets/:id' (literal), not string
```

The `satisfies` operator checks that a value matches a type *without widening it*:

```ts
const statusColors = {
  Open: 'green',
  InProgress: 'amber',
  Resolved: 'blue',
  Closed: 'gray',
} satisfies Record<TicketStatus, string>;   // error if a status is missing or misspelled

statusColors.Open;   // type 'green' (literal), still known precisely
```

`satisfies` is one of the most useful everyday features: validation of a value against a
type while keeping its precise inferred type.

---

## 4. Utility types

TypeScript ships many type functions. The ones you'll use constantly:

| Utility | Result |
|---|---|
| `Partial<T>` | All properties optional |
| `Required<T>` | All properties required |
| `Readonly<T>` | All properties readonly |
| `Pick<T, K>` | Only properties `K` |
| `Omit<T, K>` | All properties except `K` |
| `Record<K, V>` | Object with keys `K` and values `V` |
| `Exclude<U, X>` / `Extract<U, X>` | Remove / keep union members assignable to `X` |
| `NonNullable<T>` | Remove `null` and `undefined` |
| `ReturnType<F>` / `Parameters<F>` | A function's return / parameter types |
| `Awaited<T>` | Unwrap a `Promise` |

```ts
type EditableTicketFields = Pick<TicketSummary, 'title' | 'priority'>;
type TicketPatch = Partial<EditableTicketFields>;
type ActiveStatus = Exclude<TicketStatus, 'Resolved' | 'Closed'>;   // 'Open' | 'InProgress'
type ListResult = Awaited<ReturnType<typeof ticketsApi.list>>;      // Page<TicketSummary>
```

That last line is a common pattern: derive types from implementations, so changing the API
client updates every dependent type.

---

## 5. Mapped and conditional types

### Mapped types

Build a new object type by transforming each property:

```ts
type Setters<T> = {
  [K in keyof T as `set${Capitalize<string & K>}`]: (value: T[K]) => void;
};

type TicketSetters = Setters<Pick<TicketSummary, 'title' | 'priority'>>;
// { setTitle: (value: string) => void; setPriority: (value: TicketPriority) => void }
```

`as` remaps keys; template literal types build the new names.

### Conditional types

```ts
type ElementType<T> = T extends readonly (infer U)[] ? U : T;

type A = ElementType<string[]>;     // string
type B = ElementType<number>;       // number
```

`infer U` captures the array's element type. Conditional types **distribute** over unions:
`ElementType<string[] | number[]>` is `string | number`.

A practical example: the props of any React component, or the result type of an API method:

```ts
type ApiResult<K extends keyof typeof ticketsApi> = Awaited<ReturnType<(typeof ticketsApi)[K]>>;
type TicketDetail = ApiResult<'get'>;     // TicketSummary
```

### Recursive types

```ts
type DeepReadonly<T> =
  T extends (...args: never[]) => unknown ? T :
  T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> } :
  T;
```

---

## 6. Branded types: nominal typing on demand

Structural typing means a ticket ID and a user ID, both strings, are interchangeable, the
exact problem Book I, Chapter 2 solved in C# with `readonly record struct TicketId`.
TypeScript's equivalent is a **branded type**:

```ts
declare const brand: unique symbol;
type Brand<T, B extends string> = T & { readonly [brand]: B };

export type TicketId = Brand<string, 'TicketId'>;
export type UserId = Brand<string, 'UserId'>;

export function toTicketId(value: string): TicketId {
  if (!/^T-\d+$/.test(value)) throw new Error(`Invalid ticket id: ${value}`);
  return value as TicketId;           // the only place the brand is applied
}

function openTicket(id: TicketId) { /* ... */ }

openTicket(toTicketId('T-42'));       // ✓
openTicket('T-42');                   // ✗ string is not a TicketId
openTicket(someUserId);               // ✗ UserId is not a TicketId
```

At run time it's still a plain string (zero cost). At compile time, the only way to get a
`TicketId` is through the validating constructor: "parse, don't validate" again.

---

## 7. Variance, `readonly` and function types

A few subtleties that matter in real code:

- **Arrays are mutable and covariant** in TypeScript (as in C#, but without the runtime
  check): `const xs: (string | number)[] = names as string[]` is allowed, and pushing a
  number into it corrupts `names`. Use `readonly string[]` / `ReadonlyArray<T>` for
  parameters you don't mutate; it's both safer and more flexible for callers.
- **Method shorthand parameters are checked bivariantly** (a historical compromise);
  function-property syntax (`handler: (x: T) => void`) is checked strictly under
  `strictFunctionTypes`. Prefer the function-property syntax in your own types.
- **`readonly` is shallow** and compile-time only, like `const`.

---

## 8. Type design: when clever types hurt

> **🧭 When not to use advanced types:** Type-level programming is for **libraries and
> shared infrastructure**, where a complex type once saves effort in hundreds of call sites.
> In application code, prefer plain interfaces and unions. If a type takes more than a
> minute to understand, needs comments explaining how it works, or produces error messages
> nobody can read, simplify it, even at the cost of a little duplication.

Guidelines:

1. **Derive from a single source of truth** (`Pick`, `Omit`, `typeof`, `z.infer`) rather
   than duplicating shapes.
2. **Model states with discriminated unions**, not flags and optional fields (Chapter 4).
3. **Prefer inference** over annotations for locals; annotate public APIs.
4. **Keep the type-level logic in a few utility files**, documented and tested.
5. **Test complex types**: `expectTypeOf` in Vitest, or `// @ts-expect-error` lines that
   must fail.
6. **Watch compile performance**: deeply recursive conditional types can slow the type
   checker dramatically (less so with TypeScript 7's native compiler, but still real).

---

## 9. In practice: type-safe building blocks for Beacon

### Branded ticket IDs at the boundary

The API returns IDs as strings; the client brands them once, at the boundary, so the rest
of the app can't confuse them:

```ts
// web/src/api/ids.ts
declare const brand: unique symbol;
type Brand<T, B extends string> = T & { readonly [brand]: B };

export type TicketId = Brand<string, 'TicketId'>;
export const isTicketId = (v: string): v is TicketId => /^T-\d+$/.test(v);
export function asTicketId(v: string): TicketId {
  if (!isTicketId(v)) throw new Error(`Invalid ticket id "${v}"`);
  return v;
}
```

`TicketSummary.id` becomes `TicketId`, and routes parse the URL parameter with `asTicketId`
(Book VI), so an invalid ID in the URL fails at one obvious place.

### A typed result for forms

Forms need to know which fields failed validation on the server (problem details `errors`).
A small generic maps server errors onto form fields, with the field names checked against
the form's type:

```ts
// web/src/forms/serverErrors.ts
import { ApiError } from '../api/client';

export type FieldErrors<TValues> = Partial<Record<keyof TValues & string, string>>;

export function toFieldErrors<TValues extends Record<string, unknown>>(
  error: unknown,
  fields: readonly (keyof TValues & string)[],
): { fieldErrors: FieldErrors<TValues>; formError?: string } {
  if (!(error instanceof ApiError)) return { fieldErrors: {}, formError: 'Something went wrong. Please try again.' };

  const serverErrors = error.fieldErrors();            // keys like "Title" or "title"
  const fieldErrors: FieldErrors<TValues> = {};
  for (const field of fields) {
    const match = Object.entries(serverErrors).find(([k]) => k.toLowerCase() === field.toLowerCase());
    if (match?.[1][0]) fieldErrors[field] = match[1][0];
  }

  const unmatched = Object.keys(fieldErrors).length === 0;
  return { fieldErrors, ...(unmatched ? { formError: error.message } : {}) };
}
```

Usage:

```ts
type CreateTicketValues = { title: string; priority: TicketPriority; description: string };

const { fieldErrors, formError } = toFieldErrors<CreateTicketValues>(err, ['title', 'priority', 'description']);
fieldErrors.title;     // string | undefined
fieldErrors.titel;     // ✗ compile error
```

### Exhaustive status display with `satisfies`

```ts
// web/src/tickets/statusMeta.ts
import type { TicketStatus } from '../api/types';

export const statusMeta = {
  Open:       { label: 'Open',        tone: 'info' },
  InProgress: { label: 'In progress', tone: 'warning' },
  Resolved:   { label: 'Resolved',    tone: 'success' },
  Closed:     { label: 'Closed',      tone: 'neutral' },
} as const satisfies Record<TicketStatus, { label: string; tone: 'info' | 'warning' | 'success' | 'neutral' }>;
```

When a new status is added to `TICKET_STATUSES` (Chapter 4), this object fails to compile
until it's handled, the same exhaustiveness guarantee as the `switch` with `assertNever`.

### Testing a type

```ts
// web/src/api/types.test.ts
import { expectTypeOf, test } from 'vitest';
import type { ticketsApi } from './client';
import type { Page, TicketSummary } from './types';

test('list returns a page of ticket summaries', () => {
  expectTypeOf<Awaited<ReturnType<typeof ticketsApi.list>>>().toEqualTypeOf<Page<TicketSummary>>();
});
```

---

## 10. What can go wrong

- **Unreadable types** that slow down everyone who touches the code.
- **Duplicated shapes** that drift apart instead of derived types.
- **Assertions (`as`) to silence the compiler** instead of fixing the types.
- **Branded types without a validating constructor**, which just moves the unsafety.
- **Mutable array covariance** corrupting data through a wider-typed alias.
- **Compile-time slowness** from complex recursive types.
- **Types that lie**: claiming more precision than the runtime guarantees.

---

## 11. How an experienced engineer thinks about this

- **One source of truth for each shape**, everything else derived.
- **Precise types at boundaries; simple types in the middle.**
- **Type-level programming is a library tool.** Application code should read like plain
  TypeScript.
- **The compiler is a collaborator.** Let it find every place a change affects, with unions,
  `satisfies` and exhaustiveness checks.
- **If the types are fighting you, the design might be wrong**, not the compiler.

---

## 12. Check yourself

**Questions**

1. What do `keyof`, indexed access types and `typeof` each produce?
2. Write `Partial<T>` yourself as a mapped type.
3. What does `infer` do in a conditional type? Give an example.
4. What's the difference between `satisfies` and a type annotation?
5. What problem do branded types solve, and what's their run-time cost?
6. Why prefer `readonly T[]` for function parameters?
7. When should you avoid advanced type-level programming?

**Exercises**

1. Implement `Mutable<T>` (removing `readonly`) and `PickByValue<T, V>` (keys whose values
   are assignable to `V`).
2. Create `UserId` and `TeamId` branded types and refactor a function that takes three
   string IDs so they can't be swapped.
3. Write `type EventHandlers<E extends string> = { [K in E as `on${Capitalize<K>}`]: () => void }`
   and use it for ticket events.
4. Use `expectTypeOf` to test three types in Beacon's frontend.

**Interview-style questions**

- "Explain generics in TypeScript and how they differ from C# generics."
- "What are mapped and conditional types? When have you used them?"
- "How would you prevent mixing up different kinds of string IDs in TypeScript?"

---

## 13. Going deeper

- [TypeScript Handbook: Type Manipulation](https://www.typescriptlang.org/docs/handbook/2/types-from-types.html)
- Matt Pocock, [Total TypeScript](https://www.totaltypescript.com/) — tutorials and
  exercises on advanced types.
- [type-challenges](https://github.com/type-challenges/type-challenges) — puzzles for
  practicing type-level programming (fun, but remember section 8).

---

## Book V wrap-up

You now understand the language under every frontend: JavaScript's values, coercion,
closures, `this` and prototypes; the event loop and async model; the module system and
toolchain; and TypeScript from structural typing and discriminated unions to type-level
programming. Beacon has a strictly configured frontend project and a typed API client.

**Next:** [Book VI — React & Frontend](../06-react/README.md) builds Beacon's user
interface.
