# The Event Loop and Async JavaScript

JavaScript runs your code on a **single thread**. There's no `lock`, no thread pool you
manage, no `Parallel.For`. Yet a browser tab handles clicks, network responses, timers and
animations at once, and Node.js servers handle thousands of concurrent connections. The
mechanism behind both is the **event loop**.

If you understood Book I, Chapter 9 (async/await in .NET), you already know half of this
chapter: JavaScript's `async`/`await` was modeled on C#'s. The other half, the event loop
and its queues, explains why a long loop freezes the page, why `setTimeout(fn, 0)` doesn't
run immediately, and why `forEach` with `async` callbacks doesn't wait.

---

## 1. The problem: one thread, many things happening

A browser tab must:

- run JavaScript,
- respond to user input,
- render the page (ideally 60+ times per second),
- handle network responses and timers.

If JavaScript could block (wait synchronously for a network response, like `.Result` in C#),
the whole tab would freeze: no clicks, no rendering. So JavaScript's design rule is: **never
block. Start work, register a callback, return, and let the event loop call you back.**

---

## 2. The mental model: call stack, queues and the loop

```text
 ┌────────────────┐        ┌────────────────────────────────────┐
 │  Call stack    │        │  Host APIs (browser / Node.js)     │
 │  (your code,   │──────► │  timers, fetch/network, DOM events,│
 │   one frame at │ starts │  file system (Node), ...           │
 │   a time)      │        └─────────────┬──────────────────────┘
 └───────▲────────┘                      │ when done, queue a callback
         │                               ▼
         │            ┌──────────────────────────────────────────┐
         │            │ Microtask queue: promise reactions,       │  ← drained completely
         │            │ queueMicrotask, await continuations       │    after each task
         │            ├──────────────────────────────────────────┤
         └── event ───│ Task (macrotask) queue: timers, I/O,      │  ← one task per loop turn
             loop     │ events, message channel                   │
                      └──────────────────────────────────────────┘
```

The event loop repeats:

1. Take **one task** from the task queue and run it to completion (until the call stack is
   empty).
2. Run **all microtasks** in the microtask queue, including any microtasks they add.
3. (Browser) **Render** if needed: style, layout, paint.
4. Repeat.

Crucial consequences:

- **Run-to-completion.** Once a function starts, nothing else runs until it returns (or
  hits an `await`). No other code can interrupt it, so there are no data races in the C#
  sense. You don't need locks.
- **Long synchronous work blocks everything.** A loop taking 2 seconds freezes rendering and
  input for 2 seconds.
- **Timers are minimums, not guarantees.** `setTimeout(fn, 0)` queues a task; it runs after
  the current code *and all pending microtasks*, and possibly after rendering.

### Ordering quiz

```js
console.log('1 sync');
setTimeout(() => console.log('2 timeout (task)'), 0);
Promise.resolve().then(() => console.log('3 promise (microtask)'));
queueMicrotask(() => console.log('4 microtask'));
console.log('5 sync');
```

Output: `1 sync`, `5 sync`, `3 promise (microtask)`, `4 microtask`, `2 timeout (task)`.
Synchronous code first, then all microtasks, then the next task.

---

## 3. From callbacks to promises

### Callbacks

Early asynchronous JavaScript passed callbacks:

```js
getTicket('T-1', (err, ticket) => {
  if (err) return handle(err);
  getComments(ticket.id, (err, comments) => {
    if (err) return handle(err);
    getUser(comments[0].author, (err, user) => { /* ... */ });   // "callback hell"
  });
});
```

Nesting, error handling repeated at every level, and no way to compose.

### Promises

A **Promise** is JavaScript's `Task<T>`: an object representing a future value, which is
**pending**, then **fulfilled** with a value or **rejected** with an error.

```js
fetch('/api/tickets/T-1')
  .then(response => response.json())
  .then(ticket => console.log(ticket.title))
  .catch(err => console.error('Failed', err))
  .finally(() => setLoading(false));
```

Each `.then` returns a new promise, so chains are flat, and a single `.catch` handles errors
from any step. Promise callbacks always run as **microtasks**, even if the promise is already
resolved.

### `async` / `await`

The same thing, written sequentially, exactly as in C#:

```js
async function loadTicket(id) {
  try {
    const response = await fetch(`/api/tickets/${encodeURIComponent(id)}`);
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    return await response.json();
  } catch (err) {
    throw new Error(`Failed to load ticket ${id}`, { cause: err });
  }
}
```

- An `async` function **always returns a promise**.
- `await` suspends the function; the rest runs later as a microtask when the promise settles.
- Errors become rejections; `try/catch` around `await` catches them.

### C# vs JavaScript async

| | C# | JavaScript |
|---|---|---|
| Future value | `Task<T>` | `Promise<T>` |
| Starts running | When the method is called (hot) | When created (hot) |
| Where continuations run | Thread pool or captured `SynchronizationContext` | Always the single thread, via the microtask queue |
| Blocking wait | `.Result` / `.Wait()` (dangerous) | **Impossible** (no blocking API) |
| Cancellation | `CancellationToken` | `AbortController` / `AbortSignal` |
| Combinators | `Task.WhenAll`, `WhenAny` | `Promise.all`, `allSettled`, `race`, `any` |
| Unobserved failures | `TaskScheduler.UnobservedTaskException` | `unhandledrejection` event; Node.js crashes by default |

The deadlock and thread-pool-starvation problems from Book I don't exist here (there's no
way to block). The trade-off: CPU-heavy work can't be offloaded to another thread without
Web Workers (section 6).

---

## 4. Concurrency patterns

### Sequential vs concurrent

```js
// Sequential: total time = sum
const ticket = await api.getTicket(id);
const team = await api.getTeam(teamId);

// Concurrent: total time = max
const [ticket2, team2] = await Promise.all([api.getTicket(id), api.getTeam(teamId)]);
```

### Combinators

| Combinator | Resolves when | Rejects when |
|---|---|---|
| `Promise.all` | All fulfill (array of values) | **Any** rejects (fail fast) |
| `Promise.allSettled` | All settle (array of `{status, value/reason}`) | Never |
| `Promise.race` | First settles (either way) | First settles with rejection |
| `Promise.any` | First fulfills | All reject |

### The `forEach` trap

```js
// ✗ forEach ignores returned promises: "done" logs before any save completes,
//   and errors become unhandled rejections
tickets.forEach(async t => { await api.save(t); });
console.log('done');

// ✓ Sequential
for (const t of tickets) await api.save(t);

// ✓ Concurrent
await Promise.all(tickets.map(t => api.save(t)));
```

This is the same mistake as `List.ForEach(async x => ...)` creating `async void` in C#
(Book I, Chapter 9).

### Limiting concurrency

`Promise.all` over 1,000 items fires 1,000 requests at once. Limit it with a small pool:

```js
async function mapWithLimit(items, limit, fn) {
  const results = new Array(items.length);
  let next = 0;
  async function worker() {
    while (next < items.length) {
      const i = next++;                 // safe: no other code runs between these lines
      results[i] = await fn(items[i]);
    }
  }
  await Promise.all(Array.from({ length: Math.min(limit, items.length) }, worker));
  return results;
}
```

Notice `next++` needs no lock: run-to-completion guarantees no other code runs in between.

### Cancellation with `AbortController`

```js
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 5000);   // or AbortSignal.timeout(5000)

try {
  const res = await fetch('/api/tickets', { signal: controller.signal });
  // ...
} catch (err) {
  if (err.name === 'AbortError') { /* cancelled: not an error to report */ }
  else throw err;
} finally {
  clearTimeout(timeout);
}
```

Exactly the `CancellationToken` pattern. React uses it to cancel requests when a component
unmounts or a search term changes (Book VI).

---

## 5. Race conditions without threads

No threads doesn't mean no races. Asynchronous **interleaving** creates them:

```js
let currentResults = [];

async function search(term) {
  const results = await api.search(term);   // responses can arrive out of order
  currentResults = results;                 // a slow response for "vp" can overwrite "vpn"
}

input.addEventListener('input', e => search(e.target.value));
```

Typing "vpn" sends three requests; if the response for "vp" arrives last, the UI shows
results for the wrong query. Fixes: abort the previous request (`AbortController`), or
ignore stale responses by checking that the term still matches. Data-fetching libraries
(TanStack Query, Book VI) handle this for you.

> **🧱 Durable:** Any time there's an `await` between reading state and writing it, assume
> something else may have changed the state in between. That's true in JavaScript, in C#,
> and in databases (Book IV, Chapter 5).

---

## 6. Keeping the main thread free

In browsers, anything that runs longer than ~50 ms makes the page feel sluggish (a
**long task**). Options for heavy work:

- **Do less**: paginate, virtualize long lists (Book VI), debounce input handlers.
- **Yield** periodically in long loops (`await new Promise(r => setTimeout(r))`, or
  `scheduler.yield()` where supported) so input and rendering can run.
- **Web Workers**: real background threads with their own event loop, communicating by
  messages (no shared objects, so no locks). Good for parsing large files, image
  processing, heavy computation.
- **Move it to the server.**

In Node.js, the same principle applies: a CPU-heavy request handler blocks every other
request on that process. Node offloads file system and DNS work to a thread pool
internally, but your JavaScript runs on one thread; use worker threads or other processes
for CPU-bound work.

---

## 7. Node.js in brief

Node.js is JavaScript outside the browser: V8 plus an event loop (libuv) plus APIs for
files, networking and processes. For a .NET developer, it matters in three ways:

1. **Frontend tooling runs on Node**: package managers, bundlers, test runners, linters
   (next chapter).
2. **Backend-for-frontend and server-side rendering** may run on Node (Book VI).
3. **It's a common backend choice**, so you'll meet it in other teams.

Its concurrency model is exactly this chapter: one thread for JavaScript, non-blocking I/O
for everything else. That's why it handles many concurrent I/O-bound connections well and
CPU-bound work poorly.

> **🔄 Current (as of October 2026):** Node.js 24 is the Active LTS release; Node.js 26
> enters LTS in October 2026. Even-numbered releases become LTS. Node can now run
> TypeScript files directly by stripping types (`node file.ts`), though type *checking*
> still requires the TypeScript compiler.

---

## 8. In practice: an API client for Beacon

Let's write the fetch layer the React app will build on, in plain JavaScript (the next
chapters type it).

```js
// api.js
export class ApiError extends Error {
  constructor(status, problem) {
    super(problem?.title ?? `HTTP ${status}`);
    this.name = 'ApiError';
    this.status = status;
    this.problem = problem;            // RFC 9457 problem details (Book III)
  }
}

export async function request(path, { method = 'GET', body, signal, headers = {} } = {}) {
  const response = await fetch(`/api${path}`, {
    method,
    signal,
    headers: {
      Accept: 'application/json',
      ...(body !== undefined && { 'Content-Type': 'application/json' }),
      ...headers,
    },
    body: body === undefined ? undefined : JSON.stringify(body),
    credentials: 'include',            // send the BFF session cookie (Book VI, Chapter 5)
  });

  if (response.status === 204) return undefined;

  const isJson = response.headers.get('Content-Type')?.includes('json');
  const payload = isJson ? await response.json() : await response.text();

  if (!response.ok) throw new ApiError(response.status, isJson ? payload : undefined);
  return payload;
}

export const tickets = {
  list: (params, signal) => request(`/tickets?${new URLSearchParams(params)}`, { signal }),
  get: (id, signal) => request(`/tickets/${encodeURIComponent(id)}`, { signal }),
  create: (input) => request('/tickets', {
    method: 'POST',
    body: input,
    headers: { 'Idempotency-Key': crypto.randomUUID() },   // Book III, Chapter 4
  }),
  addComment: (id, body) => request(`/tickets/${encodeURIComponent(id)}/comments`, { method: 'POST', body: { body } }),
};
```

Design notes:

- **Errors become `ApiError`** carrying the status and problem details, so UI code can show
  field-level validation errors or a friendly message, and branch on `status`.
- **Every call accepts an `AbortSignal`**, so components can cancel requests.
- **`encodeURIComponent`** on path segments; `URLSearchParams` for query strings. Never
  concatenate user input into URLs raw.
- **`fetch` doesn't reject on HTTP errors** (only on network failures), a common surprise:
  checking `response.ok` is required.
- **Idempotency keys** generated per create, so a retried create doesn't duplicate tickets.

A search box using it, with stale-response protection:

```js
let controller;

async function onSearchInput(term) {
  controller?.abort();                       // cancel the previous request
  controller = new AbortController();
  try {
    const page = await tickets.list({ q: term, limit: 20 }, controller.signal);
    render(page.items);
  } catch (err) {
    if (err.name !== 'AbortError') showError(err);
  }
}
```

---

## 9. What can go wrong

- **Long synchronous work** freezing the UI.
- **`forEach` with async callbacks** not waiting and losing errors.
- **Unhandled promise rejections** (missing `await` or `.catch`).
- **Sequential awaits** where requests could run concurrently, and unbounded `Promise.all`
  where they shouldn't.
- **Out-of-order responses** overwriting newer state.
- **Forgetting `fetch` resolves on HTTP 4xx/5xx.**
- **No cancellation**: requests continuing after the user navigated away, then updating
  unmounted UI.

---

## 10. How an experienced engineer thinks about this

- **The main thread is precious.** Keep tasks short; move heavy work elsewhere.
- **Every `await` is a point where the world can change.** Re-check assumptions after it.
- **Cancel what you no longer need.** `AbortController` everywhere requests are started.
- **Choose concurrency deliberately**: sequential, `all`, `allSettled` or limited pools.
- **The same async model as C#, minus threads**: no deadlocks, but no blocking escape
  hatch either.

---

## 11. Check yourself

**Questions**

1. Describe the event loop. What's the difference between tasks and microtasks?
2. Why doesn't `setTimeout(fn, 0)` run immediately?
3. Why don't you need locks in JavaScript? Can you still have race conditions?
4. What's the difference between `Promise.all` and `Promise.allSettled`?
5. Why doesn't `forEach` work with `async` callbacks?
6. How do you cancel a `fetch`?
7. How does JavaScript's async model differ from .NET's?

**Exercises**

1. Predict the output order of a snippet mixing `setTimeout`, `Promise.then`,
   `queueMicrotask` and `await`, then run it.
2. Write a `retry(fn, { attempts, baseDelayMs, signal })` helper with exponential back-off
   and jitter that respects cancellation.
3. Freeze a page with a 3-second loop, then make it responsive by processing in chunks
   with yielding.
4. Reproduce the out-of-order search results bug with a fake API that adds random delays,
   then fix it two ways.

**Interview-style questions**

- "Explain the JavaScript event loop."
- "What's the output of this code?" (A task/microtask ordering puzzle.)
- "How would you limit the number of concurrent requests?"
- "How is `async/await` in JavaScript related to Promises?"

---

## 12. Going deeper

- [MDN: Using promises](https://developer.mozilla.org/docs/Web/JavaScript/Guide/Using_promises)
  and [The event loop](https://developer.mozilla.org/docs/Web/JavaScript/Event_loop)
- Jake Archibald, "In The Loop" (JSConf talk) — a visual explanation of tasks, microtasks
  and rendering.
- [Node.js docs: The event loop, timers and process.nextTick](https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick)

**Next:** [Chapter 3 — Modules, Tooling and the Ecosystem](03-modules-tooling-and-the-ecosystem.md)
covers how JavaScript code is organized, installed and built.
