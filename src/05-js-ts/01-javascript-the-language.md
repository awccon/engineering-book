# JavaScript the Language

You'll write TypeScript, not plain JavaScript, for Beacon's frontend. So why a whole
chapter on JavaScript? Because **TypeScript is JavaScript with types erased at build
time**. Every TypeScript program runs as JavaScript, with JavaScript's semantics: its
equality rules, its `this`, its closures, its prototypes, its numbers. TypeScript catches
many mistakes, but when something behaves strangely at run time, the explanation is always
in JavaScript.

This chapter is written for a C# developer. It focuses on where JavaScript differs from
C#, because those differences are where the bugs are.

---

## 1. The problem: a language designed in ten days, used everywhere

JavaScript was created in 1995, in about ten days, to add small interactions to web pages.
It became the only language that runs natively in every browser, then spread to servers
(Node.js), mobile apps, desktop apps and edge functions. It has evolved enormously (the
yearly ECMAScript standards since 2015 added classes, modules, `async`/`await` and much
more), but **it never breaks backward compatibility**. Every early design decision is
still there, alongside the modern alternatives.

So modern JavaScript is two languages: a good modern one, and a set of legacy behaviors
you must recognize and avoid.

---

## 2. The mental model: dynamic, prototype-based, single-threaded

Compared to C#:

| | C# | JavaScript |
|---|---|---|
| Typing | Static, nominal | **Dynamic**: values have types; variables don't |
| Compilation | Ahead of time to IL, then JIT | Parsed and JIT-compiled by the engine (V8, SpiderMonkey, JavaScriptCore) at run time |
| Objects | Instances of classes | **Bags of properties** linked to **prototypes** |
| Functions | Methods on types (and delegates) | **First-class values**; objects themselves |
| Concurrency | Threads + async | **Single-threaded event loop** + async (next chapter) |
| Numbers | `int`, `long`, `double`, `decimal`... | One `number` type (64-bit float) + `bigint` |
| Null | `null` | **Two**: `null` and `undefined` |

---

## 3. Values and types

JavaScript has seven **primitive** types and objects:

| Type | Examples | Notes |
|---|---|---|
| `number` | `42`, `3.14`, `NaN`, `Infinity` | All numbers are 64-bit floats (like C# `double`) |
| `bigint` | `9007199254740993n` | Arbitrary-precision integers |
| `string` | `'a'`, `"b"`, `` `c${x}` `` | Immutable, UTF-16 |
| `boolean` | `true`, `false` | |
| `undefined` | `undefined` | "No value was ever assigned" |
| `null` | `null` | "Intentionally empty" |
| `symbol` | `Symbol('id')` | Unique keys, rarely used directly |
| `object` | `{}`, `[]`, functions, dates, maps... | Everything else; passed by reference |

### Numbers are doubles

```js
0.1 + 0.2;                    // 0.30000000000000004
Number.MAX_SAFE_INTEGER;      // 9007199254740991 (2^53 - 1)
9007199254740993 === 9007199254740992;   // true!
```

Consequences for a .NET backend developer:

- **Never use JavaScript numbers for money calculations.** Use integer minor units (cents)
  or a decimal library, or let the server calculate.
- **64-bit IDs lose precision.** A C# `long` like `9007199254740993` serialized as a JSON
  number becomes a different number in JavaScript. Serialize large IDs as strings (Beacon's
  `"T-42"` IDs avoid this entirely; Book III, Chapter 4).

### `null` and `undefined`

`undefined` is what you get for missing things: an unassigned variable, a missing property,
a function that returns nothing, a missing argument. `null` is an explicit "nothing."
APIs and codebases mix them. Modern operators handle both:

```js
const name = user?.profile?.displayName ?? 'Anonymous';
//               ?. stops at null/undefined   ?? defaults only for null/undefined
```

Note `??` vs `||`: `count || 10` replaces `0` with `10` (because `0` is falsy);
`count ?? 10` keeps `0`.

---

## 4. Equality and coercion

JavaScript converts types implicitly in many operations. This is the source of its most
famous oddities:

```js
'5' + 1;        // '51'   (number converted to string)
'5' - 1;        // 4      (string converted to number)
[] + {};        // '[object Object]'
'' == 0;        // true
'0' == 0;       // true
'' == '0';      // false  (== isn't even transitive)
null == undefined;  // true
NaN === NaN;    // false  (use Number.isNaN)
```

The rule that avoids nearly all of it: **always use `===` and `!==`** (strict equality, no
coercion). The single common exception is `x == null`, an idiom for "null or undefined."

### Truthiness

In conditions, values are coerced to booleans. **Falsy** values: `false`, `0`, `-0`, `0n`,
`''`, `null`, `undefined`, `NaN`. Everything else is **truthy**, including `'0'`, `'false'`,
`[]` and `{}`.

```js
if (items.length) { ... }          // fine: 0 is falsy
if (user.age) { ... }              // bug: age 0 is treated as missing
if (user.age !== undefined) { ... }  // explicit
```

### Object equality is reference equality

```js
({ id: 1 }) === ({ id: 1 });   // false: different objects
```

Like C# classes (not records), objects compare by reference. There's no built-in value
equality for objects; React's re-rendering rules (Book VI) depend on exactly this.

---

## 5. Variables and scope

```js
const ticket = { id: 'T-1' };   // can't be reassigned (the object can still be mutated)
let count = 0;                  // can be reassigned
var legacy = 1;                 // ✗ function-scoped, hoisted; don't use
```

- **`const` by default, `let` when you need to reassign, never `var`.**
- `const` prevents rebinding the *variable*, not mutating the *object*, like a C# `readonly`
  reference field. `Object.freeze` makes an object shallowly immutable.
- `let` and `const` are **block-scoped** (like C# locals). `var` is function-scoped and
  *hoisted*, which caused the classic loop-closure bug (the same one as C#'s `for` loop in
  Book I, Chapter 6, but worse).

### Closures

Closures work like C#'s: a function captures the variables of its surrounding scope, not
their values:

```js
function makeCounter() {
  let count = 0;
  return () => ++count;
}
const next = makeCounter();
next(); // 1
next(); // 2
```

React hooks (Book VI) are built entirely on closures, and the most confusing React bugs
("stale closures") are closures capturing an old value.

---

## 6. Objects and arrays

### Objects are dynamic property bags

```js
const ticket = { id: 'T-1', title: 'VPN drops', priority: 'High' };
ticket.status = 'Open';              // add a property
delete ticket.priority;              // remove one
ticket['title'];                     // bracket access with a string key
const { id, title, ...rest } = ticket;   // destructuring with rest
const copy = { ...ticket, status: 'Resolved' };   // spread: shallow copy with changes
```

The spread pattern `{ ...obj, changed: value }` is JavaScript's equivalent of C#'s
`record with { ... }`, and it's how state is updated immutably in React.

> **⚠️ What can go wrong:** Spread is a **shallow** copy. `{ ...ticket }` copies the
> reference to `ticket.comments`, not the array. Mutating `copy.comments.push(...)` mutates
> the original too. Copy nested structures explicitly, or use `structuredClone` for a deep copy.

### Arrays

Arrays are objects with numeric keys and many methods. The functional ones are JavaScript's
LINQ:

| LINQ | JavaScript | Mutates? |
|---|---|---|
| `Where` | `filter` | No |
| `Select` | `map` | No |
| `SelectMany` | `flatMap` | No |
| `Aggregate` | `reduce` | No |
| `First(pred)` | `find` | No |
| `Any` / `All` | `some` / `every` | No |
| `OrderBy` | `toSorted` (ES2023) / `sort` | `sort` **mutates in place** |
| `Reverse` | `toReversed` / `reverse` | `reverse` mutates |
| `GroupBy` | `Object.groupBy` / `Map.groupBy` (ES2024) | No |

```js
const urgentTitles = tickets
  .filter(t => t.priority === 'Urgent' && t.status !== 'Closed')
  .toSorted((a, b) => a.createdAt.localeCompare(b.createdAt))
  .map(t => t.title);
```

Unlike LINQ, array methods are **eager**: each creates a new array immediately.

Two traps: `sort()` mutates the array (and sorts numbers as strings by default:
`[10, 9, 1].sort()` → `[1, 10, 9]`), and `forEach` doesn't wait for `async` callbacks
(next chapter).

### `Map` and `Set`

Use `Map` for dictionaries with non-string keys or frequent additions and removals, and
`Set` for unique values. Plain objects work as string-keyed dictionaries but inherit
prototype properties and have quirks with keys like `__proto__`.

---

## 7. Functions and `this`

Functions are values: passed, returned, stored in objects and arrays. There are two main
syntaxes:

```js
function add(a, b) { return a + b; }    // function declaration
const add2 = (a, b) => a + b;           // arrow function
```

The important difference is **`this`**:

- In a regular `function`, `this` is decided **at call time** by how the function is called:
  `obj.method()` sets `this = obj`; calling the same function detached (`const m = obj.method; m();`)
  sets `this` to `undefined` (in strict mode).
- **Arrow functions don't have their own `this`**; they use the `this` of the scope where
  they're defined, like a closure.

```js
class Poller {
  intervalMs = 5000;
  start() {
    setInterval(function () { this.poll(); }, this.intervalMs);   // ✗ this is undefined in the callback
    setInterval(() => this.poll(), this.intervalMs);              // ✓ arrow captures this
  }
  poll() { /* ... */ }
}
```

This is radically different from C#, where `this` is always the instance. In modern code
the practical rule is simple: **use arrow functions for callbacks**, and you'll rarely
think about `this`. React function components avoid `this` entirely.

---

## 8. Prototypes and classes

JavaScript objects inherit from other objects via a **prototype chain**: if a property isn't
found on an object, the engine looks at its prototype, then the prototype's prototype, and
so on.

```js
const base = { greet() { return `Hello from ${this.name}`; } };
const obj = Object.create(base);
obj.name = 'Beacon';
obj.greet();   // found on the prototype
```

The `class` syntax (ES2015) is a cleaner way to set up prototypes. It looks like C#:

```js
class TicketStore {
  #tickets = new Map();               // # = truly private field

  add(ticket) { this.#tickets.set(ticket.id, ticket); }
  get count() { return this.#tickets.size; }
  static empty() { return new TicketStore(); }
}

class AuditedTicketStore extends TicketStore {
  add(ticket) {
    console.log('adding', ticket.id);
    super.add(ticket);
  }
}
```

But underneath, it's still prototypes: methods live on `TicketStore.prototype`, and
everything is dynamic at run time. In modern frontend code, classes are used less than in
C#; plain objects, functions and modules do most of the work.

---

## 9. Errors

```js
try {
  JSON.parse('{bad json');
} catch (err) {
  console.error(err.message);
} finally {
  cleanup();
}
```

Differences from C#:

- **You can throw anything**: `throw 'oops'`, `throw { code: 42 }`. Catch blocks receive
  `unknown` values. Always throw `Error` objects (or subclasses), which carry a stack trace.
- **No typed catch clauses.** Check inside: `if (err instanceof TypeError) ...`.
- **`Error.cause`** links errors, like C#'s `InnerException`:
  `throw new Error('Failed to load ticket', { cause: err })`.

---

## 10. In practice: JavaScript for Beacon

Beacon's frontend will be TypeScript, but let's write a small plain-JavaScript module and
run it with Node.js, to see these behaviors directly.

```bash
node --version          # Node.js 24 LTS or later
mkdir beacon-js-playground && cd beacon-js-playground
```

```js
// tickets.js
const tickets = [
  { id: 'T-1', title: 'Cannot log in', priority: 'High', status: 'Open', createdAt: '2026-10-07T09:00:00Z', comments: [] },
  { id: 'T-2', title: 'VPN drops', priority: 'Urgent', status: 'InProgress', createdAt: '2026-10-07T08:15:00Z', comments: [{ author: 'maria' }] },
  { id: 'T-3', title: 'Typo on pricing page', priority: 'Low', status: 'Resolved', createdAt: '2026-10-06T16:40:00Z', comments: [] },
];

const priorityRank = { Low: 0, Normal: 1, High: 2, Urgent: 3 };

// Triage order: most urgent first, then oldest first (Book I's TriageQueue, in JS)
const queue = tickets
  .filter(t => t.status === 'Open' || t.status === 'InProgress')
  .toSorted((a, b) =>
    priorityRank[b.priority] - priorityRank[a.priority] ||
    a.createdAt.localeCompare(b.createdAt));

console.log(queue.map(t => `${t.id} ${t.priority} ${t.title}`));

// Immutable update, React-style
const resolved = { ...tickets[0], status: 'Resolved' };
console.log(tickets[0].status, resolved.status);    // Open Resolved

// The shallow-copy trap
const copy = { ...tickets[1] };
copy.comments.push({ author: 'omar' });
console.log(tickets[1].comments.length);           // 2: the original changed too!

// Grouping (ES2024)
console.log(Object.groupBy(tickets, t => t.status));
```

```bash
node tickets.js
```

Notice:

- `||` chains comparisons: when priorities are equal, the difference is `0` (falsy), so the
  second comparison decides. A compact JavaScript idiom.
- ISO 8601 UTC timestamps sort correctly as strings, which is one more reason the API
  returns them in that format (Book III, Chapter 4).
- The shallow-copy trap appears exactly as described.

---

## 11. What can go wrong

- **`==` coercion surprises.** Use `===`.
- **Floating-point arithmetic for money; large integer IDs losing precision.**
- **Falsy checks treating `0` and `''` as missing.**
- **`this` lost in callbacks** with regular functions.
- **Shallow copies** sharing nested objects.
- **`sort()` mutating arrays** and sorting numbers as strings.
- **Throwing non-`Error` values**, losing stack traces.
- **Typos creating new properties** instead of errors (`ticket.titel = 'x'` succeeds
  silently). TypeScript fixes this one.

---

## 12. How an experienced engineer thinks about this

- **Use the modern subset**: `const`/`let`, `===`, arrow functions, `?.` and `??`, spread,
  modules, classes with `#private`, non-mutating array methods.
- **Know the legacy behaviors** well enough to recognize them in bugs and old code.
- **Treat data as immutable** by convention; it makes UI frameworks and debugging simpler.
- **Let TypeScript catch what it can**, and remember it can't change run-time semantics.

---

## 13. Check yourself

**Questions**

1. Why is `0.1 + 0.2 !== 0.3` in JavaScript, and what does it mean for money?
2. What's the difference between `null` and `undefined`? Between `??` and `||`?
3. Why should you use `===` instead of `==`?
4. List the falsy values. Is `[]` truthy?
5. How does `this` differ between regular functions and arrow functions?
6. What's a shallow copy, and how can it cause bugs?
7. What does `class` really create under the hood?

**Exercises**

1. Predict, then verify in Node: `[] == false`, `null >= 0`, `'b' + 'a' + +'a' + 'a'`.
2. Implement Book I's `AgentWorkloadReport` in JavaScript with `Object.groupBy`, `filter`
   and `reduce`.
3. Write a function `deepFreeze(obj)` and use it to catch accidental mutation.
4. Serialize `{ id: 9007199254740993 }` with `JSON.stringify` after parsing it from a string,
   and explain the result.

**Interview-style questions**

- "Explain closures in JavaScript."
- "What's the difference between `==` and `===`?"
- "How does `this` work in JavaScript?"
- "What's prototypal inheritance?"

---

## 14. Going deeper

- [MDN JavaScript Guide](https://developer.mozilla.org/docs/Web/JavaScript/Guide) — the
  best reference.
- Kyle Simpson, *You Don't Know JS Yet* (free on GitHub) — deep dives into scope,
  closures, `this` and types.
- [javascript.info](https://javascript.info/) — a thorough modern tutorial.

**Next:** [Chapter 2 — The Event Loop and Async JavaScript](02-the-event-loop-and-async-javascript.md)
explains how a single-threaded language handles concurrency.
