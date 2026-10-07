# Hooks

Hooks are how function components remember things, react to changes and connect to the
outside world. `useState` and `useEffect` look simple, and most React developers use them
daily. They're also the source of the most common React bugs: effects that run in infinite
loops, stale values inside callbacks, race conditions in data fetching, and effects that
exist only to copy one piece of state into another.

This chapter explains how hooks work underneath, then covers each important hook with the
rules and judgment that come with it. It ends with custom hooks for Beacon.

---

## 1. The problem: functions that need memory

A component is a function called on every render. Ordinary local variables are recreated
each time, so a function can't remember anything between renders, and can't schedule work
after the DOM updates. Hooks provide those capabilities: state that survives renders,
effects that run after commits, references to DOM nodes, and memoized values.

---

## 2. The mental model: hooks are slots, matched by call order

React keeps, for each mounted component, a list of **hook slots**. During a render, each
hook call reads the next slot:

```text
 TicketDetail (mounted instance)
 ┌───────────────────────────────┐
 │ slot 0: useState → editing    │ ◄── 1st hook call in the function
 │ slot 1: useState → draft      │ ◄── 2nd hook call
 │ slot 2: useEffect → deps, fn  │ ◄── 3rd hook call
 │ slot 3: useRef → { current }  │ ◄── 4th hook call
 └───────────────────────────────┘
```

Slots are matched **by order**, not by name. That's why the **Rules of Hooks** exist:

1. **Only call hooks at the top level** of a component or custom hook, never inside
   conditions, loops or after early returns. If the order changes between renders, slots
   get mismatched.
2. **Only call hooks from React functions** (components and custom hooks), not ordinary
   functions.

The `eslint-plugin-react-hooks` rules (Book V, Chapter 3) enforce both. Treat its warnings
as errors.

### Every render has its own values

Each render is a separate function call with its own props, state and closures:

```tsx
function Counter() {
  const [count, setCount] = useState(0);
  function handleClick() {
    setTimeout(() => alert(count), 3000);   // alerts the count from THIS render
  }
  // ...
}
```

Click at count 0, then increment twice quickly: the alert shows `0`. The callback closed
over the `count` constant of the render in which it was created (Book V, Chapter 1's
closures). This is the "stale closure" behavior, and once you see renders as snapshots,
it's not surprising.

---

## 3. `useState`

```tsx
const [draft, setDraft] = useState('');
const [filters, setFilters] = useState<Filters>(() => readFiltersFromUrl());   // lazy initializer
```

- The initializer runs only on the first render. Pass a function for expensive initial
  values.
- **Functional updates** use the latest state, avoiding stale values when the update
  depends on the previous state:

```tsx
setCount(c => c + 1);                                   // ✓ always correct
setTickets(prev => [newTicket, ...prev]);               // ✓
setCount(count + 1); setCount(count + 1);               // ✗ both use the same snapshot: +1, not +2
```

- Setting state to the **same value** (`Object.is`) skips the re-render.

### `useReducer`

When state has several related fields and transitions with rules, a reducer centralizes
them (the same idea as Book I, Chapter 3's encapsulated entity):

```tsx
type EditorState =
  | { mode: 'viewing' }
  | { mode: 'editing'; draft: string }
  | { mode: 'saving'; draft: string }
  | { mode: 'error'; draft: string; message: string };

type EditorAction =
  | { type: 'edit'; initial: string }
  | { type: 'change'; value: string }
  | { type: 'save' }
  | { type: 'saved' }
  | { type: 'failed'; message: string }
  | { type: 'cancel' };

function editorReducer(state: EditorState, action: EditorAction): EditorState {
  switch (action.type) {
    case 'edit':   return { mode: 'editing', draft: action.initial };
    case 'change': return state.mode === 'editing' || state.mode === 'error' ? { mode: 'editing', draft: action.value } : state;
    case 'save':   return state.mode === 'editing' ? { mode: 'saving', draft: state.draft } : state;
    case 'saved':  return { mode: 'viewing' };
    case 'failed': return state.mode === 'saving' ? { mode: 'error', draft: state.draft, message: action.message } : state;
    case 'cancel': return { mode: 'viewing' };
  }
}

const [state, dispatch] = useReducer(editorReducer, { mode: 'viewing' });
```

The reducer is a pure function: trivially unit-testable, and illegal states (saving
without a draft) are unrepresentable thanks to the discriminated union.

---

## 4. `useEffect`: synchronizing with the outside world

An **effect** runs **after** React commits a render to the screen. Its purpose is to
**synchronize** the component with something outside React: a network connection, a
subscription, a timer, a browser API, a non-React widget.

```tsx
useEffect(() => {
  const connection = hub.watch(ticketId);         // start synchronizing
  return () => connection.stop();                  // cleanup: stop synchronizing
}, [ticketId]);                                    // re-synchronize when ticketId changes
```

The model: **"while this component is on screen with these dependency values, keep this
external thing in sync."** When dependencies change, React runs the cleanup for the old
values, then the effect for the new values. On unmount, it runs the cleanup.

### Dependencies

The dependency array lists every reactive value (props, state, values derived from them)
the effect uses:

- `[a, b]`: run after mount and whenever `a` or `b` changes (by `Object.is`).
- `[]`: run once after mount (and cleanup on unmount).
- No array: run after every render (rarely what you want).

**Never lie about dependencies** to control when an effect runs. Omitting a value the
effect uses means it reads stale values. The lint rule catches this. If an effect re-runs
too often, change the code so it doesn't *need* the dependency (functional updates, moving
objects inside the effect, or not using an effect at all).

### Strict Mode runs effects twice in development

In development, `<StrictMode>` mounts components, unmounts them, and mounts them again,
so effects run, clean up, and run again. This is deliberate: it exposes effects without
proper cleanup (duplicate subscriptions, double requests). If your effect breaks when run
twice, it would break in production too, when users navigate away and back.

### Data fetching in an effect (and its pitfalls)

```tsx
useEffect(() => {
  const controller = new AbortController();
  setState({ status: 'loading' });
  ticketsApi.get(ticketId, controller.signal)
    .then(data => setState({ status: 'success', data }))
    .catch(error => {
      if (error.name !== 'AbortError') setState({ status: 'error', error });
    });
  return () => controller.abort();         // cancel when ticketId changes or on unmount
}, [ticketId]);
```

This handles the race condition from Book V, Chapter 2 (a slow response for an old
`ticketId` can't overwrite the new one, because it's aborted). But hand-written fetching
still lacks caching, deduplication, retries, background refresh and shared state between
components. That's why real apps use a data-fetching library or framework loaders
(Chapter 4). Fetching in effects is fine to understand; it's rarely what you should ship.

---

## 5. You might not need an effect

The most common React mistake is using effects for things that aren't synchronization with
an external system:

```tsx
// ✗ Derived state via effect: an extra render, and a moment of inconsistent UI
const [visible, setVisible] = useState<TicketSummary[]>([]);
useEffect(() => { setVisible(tickets.filter(t => t.status === status)); }, [tickets, status]);

// ✓ Compute during render
const visible = tickets.filter(t => t.status === status);
```

```tsx
// ✗ Reacting to a user action in an effect
useEffect(() => { if (submitted) { api.save(form); setSubmitted(false); } }, [submitted]);

// ✓ Do it in the event handler that caused it
async function handleSubmit() { await api.save(form); }
```

```tsx
// ✗ Resetting state when a prop changes
useEffect(() => { setDraft(''); }, [ticketId]);

// ✓ Reset by key from the parent
<CommentBox key={ticketId} ticketId={ticketId} />
```

Ask: **"Is this effect synchronizing with something outside React?"** If not, it probably
belongs in render (derivation) or in an event handler (user action).

> **🧭 When not to use `useEffect`:** For transforming data for rendering, handling user
> events, resetting state on prop changes, or notifying parents about changes. Effects are
> for subscriptions, timers, manual DOM integration, analytics on screen view, and
> network connections (preferably through a library).

---

## 6. Refs, memoization and other hooks

### `useRef`

A ref is a mutable box (`{ current }`) that persists across renders and **doesn't trigger
re-renders** when changed. Two uses:

```tsx
const inputRef = useRef<HTMLInputElement>(null);
<input ref={inputRef} />;
inputRef.current?.focus();                 // 1. access a DOM node

const timerId = useRef<number | null>(null);   // 2. remember a non-rendering value (timer id, previous value)
```

Don't read or write refs during render (except for initialization); it breaks purity.

### `useMemo`, `useCallback` and `memo`

- `useMemo(() => compute(a, b), [a, b])` caches a computed value between renders.
- `useCallback(fn, deps)` caches a function identity.
- `memo(Component)` skips re-rendering a component if its props are shallowly equal.

They exist for performance: avoiding expensive recomputation, and keeping props stable so
memoized children can skip rendering.

> **🔄 Current (as of October 2026):** With the **React Compiler** enabled (a build-time
> Babel/SWC/Vite plugin), React automatically memoizes components and values, and most
> manual `useMemo`, `useCallback` and `memo` become unnecessary. New projects should enable
> it; existing code with manual memoization keeps working. Chapter 6 covers performance in
> depth.

### Other hooks worth knowing

| Hook | Purpose |
|---|---|
| `useContext` | Read a context value (Chapter 3) |
| `useId` | Stable unique IDs for accessibility attributes (`htmlFor`, `aria-describedby`) |
| `useTransition` / `useDeferredValue` | Mark updates as non-urgent so typing stays responsive (Chapter 6) |
| `useLayoutEffect` | Like `useEffect` but before the browser paints; for measuring layout (rare) |
| `useSyncExternalStore` | Subscribe to an external store (browser APIs, non-React state) correctly |
| `use` (React 19) | Read a promise or context during render, with Suspense |
| `useActionState`, `useOptimistic`, `useFormStatus` (React 19) | Form actions and optimistic UI (Chapter 4) |
| `useEffectEvent` (React 19.2) | An event-like function inside effects that always sees the latest props/state without being a dependency |

---

## 7. Custom hooks

A **custom hook** is a function whose name starts with `use` and that calls other hooks.
It's how you extract and reuse stateful logic (not UI) across components:

```tsx
export function useDebouncedValue<T>(value: T, delayMs = 300): T {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delayMs);
    return () => clearTimeout(id);
  }, [value, delayMs]);
  return debounced;
}
```

Each component calling a custom hook gets **its own** independent state; hooks share
*logic*, not *state*.

Good custom hooks:

- Have a clear, single purpose (`useOnlineStatus`, `useTicketSubscription`).
- Hide effects and subscriptions behind a simple interface.
- Are named for **what** they provide, not how (`useCurrentUser`, not `useFetchMeEffect`).

---

## 8. In practice: Beacon's hooks

### Live ticket updates via SignalR

Book III, Chapter 8 built `TicketHub`. A custom hook connects the React app to it, using
an effect for exactly what effects are for: synchronizing with an external connection.

```bash
pnpm add @microsoft/signalr
```

```ts
// web/src/realtime/connection.ts
import { HubConnectionBuilder, LogLevel, type HubConnection } from '@microsoft/signalr';

let connection: HubConnection | null = null;
let starting: Promise<void> | null = null;

export function getTicketHub(): HubConnection {
  connection ??= new HubConnectionBuilder()
    .withUrl('/hubs/tickets')                 // proxied by Vite in development, same origin in production
    .withAutomaticReconnect()
    .configureLogging(LogLevel.Warning)
    .build();
  return connection;
}

export async function ensureStarted(): Promise<HubConnection> {
  const hub = getTicketHub();
  if (hub.state === 'Disconnected') {
    starting ??= hub.start().finally(() => { starting = null; });
    await starting;
  }
  return hub;
}
```

```tsx
// web/src/realtime/useTicketUpdates.ts
import { useEffect, useEffectEvent } from 'react';
import type { TicketSummary } from '../api/types';
import { ensureStarted, getTicketHub } from './connection';

export function useTicketUpdates(ticketId: string, onUpdate: (ticket: TicketSummary) => void) {
  // Always calls the latest onUpdate without making it an effect dependency
  const handleUpdate = useEffectEvent((ticket: TicketSummary) => {
    if (ticket.id === ticketId) onUpdate(ticket);
  });

  useEffect(() => {
    let cancelled = false;
    const hub = getTicketHub();
    const listener = (t: TicketSummary) => handleUpdate(t);

    hub.on('TicketUpdated', listener);
    ensureStarted()
      .then(h => (cancelled ? undefined : h.invoke('Watch', ticketId)))
      .catch(err => console.warn('Live updates unavailable', err));

    return () => {
      cancelled = true;
      hub.off('TicketUpdated', listener);
      if (hub.state === 'Connected') hub.invoke('Unwatch', ticketId).catch(() => {});
    };
  }, [ticketId]);
}
```

Points:

- **The effect depends only on `ticketId`.** Navigating to another ticket unsubscribes from
  the old one and subscribes to the new one; unmounting cleans up.
- **`useEffectEvent`** lets the callback use the latest `onUpdate` without re-subscribing
  every time the parent passes a new function.
- **Strict Mode's double mount** subscribes, unsubscribes and subscribes again, which
  works because cleanup is complete.
- **Failure is non-fatal**: the page works without live updates.

In Chapter 4, `onUpdate` will update the data-fetching cache, so every component showing
that ticket refreshes.

### Debounced search

```tsx
// web/src/tickets/TicketSearch.tsx
import { useId, useState } from 'react';
import { useDebouncedValue } from '../hooks/useDebouncedValue';

export function TicketSearch({ onSearch }: { onSearch: (term: string) => void }) {
  const [term, setTerm] = useState('');
  const debounced = useDebouncedValue(term.trim(), 300);
  const inputId = useId();

  // Notifying the parent about a debounced value is legitimately a "synchronize" step,
  // but simpler: let the parent read the debounced value. Here we pass it up explicitly.
  useNotifyOnChange(debounced, onSearch);

  return (
    <div className="search">
      <label htmlFor={inputId}>Search tickets</label>
      <input id={inputId} type="search" value={term} onChange={e => setTerm(e.target.value)} />
    </div>
  );
}
```

On reflection, `useNotifyOnChange` is an effect that exists only to call a parent callback,
which section 5 warned against. A cleaner design: the parent owns `term`, and *it* uses
`useDebouncedValue(term)` to drive the query (Chapter 4 does exactly this). Code review
catches this kind of thing; it's worth practicing on your own code.

### A tested reducer

The editor reducer from section 3 is a pure function, so its tests need no React at all:

```ts
// web/src/tickets/editorReducer.test.ts
import { describe, expect, it } from 'vitest';
import { editorReducer } from './editorReducer';

describe('editorReducer', () => {
  it('ignores save while viewing', () => {
    expect(editorReducer({ mode: 'viewing' }, { type: 'save' })).toEqual({ mode: 'viewing' });
  });

  it('keeps the draft when saving fails', () => {
    const saving = { mode: 'saving', draft: 'Updated title' } as const;
    expect(editorReducer(saving, { type: 'failed', message: 'Conflict' }))
      .toEqual({ mode: 'error', draft: 'Updated title', message: 'Conflict' });
  });
});
```

---

## 9. What can go wrong

- **Conditional hook calls**, breaking slot order.
- **Stale closures** in timers, intervals and subscriptions.
- **Missing or fake dependencies**; infinite loops from objects or functions recreated
  every render in the dependency list.
- **Effects for derived state and user events.**
- **Missing cleanup**: duplicate subscriptions, leaked timers, updates after unmount.
- **Fetching in effects without cancellation**: race conditions.
- **Over-memoization** cluttering code (and, without the compiler, under-memoization
  causing slow renders).
- **Custom hooks that hide too much**, making data flow hard to follow.

---

## 10. How an experienced engineer thinks about this

- **Renders are snapshots.** Values in a render never change; new renders get new values.
- **Effects synchronize with external systems**, and every effect has a matching cleanup.
- **Prefer no effect.** Derive during render; act in event handlers; reset with keys.
- **Make illegal states unrepresentable** with reducers and discriminated unions.
- **Extract custom hooks for reusable stateful logic**, named for what they provide.

---

## 11. Check yourself

**Questions**

1. Why must hooks be called in the same order on every render?
2. What's a stale closure, and how do functional updates avoid one?
3. What is `useEffect` for? When does it run, and when does its cleanup run?
4. Why does Strict Mode run effects twice in development?
5. Give three things people often do with effects that don't need effects.
6. What's the difference between `useRef` and `useState`?
7. What does the React Compiler change about `useMemo` and `useCallback`?

**Exercises**

1. Write `useInterval(callback, delayMs)` that always calls the latest callback without
   restarting the interval.
2. Write `useLocalStorageState<T>(key, initial)` with JSON parsing guarded by a type guard
   (untrusted data, Book V, Chapter 4).
3. Find an effect in a codebase you know that could be replaced by derivation or an event
   handler, and refactor it.
4. Implement the ticket editor with `useReducer` and test every transition.

**Interview-style questions**

- "Explain the rules of hooks and why they exist."
- "What's the difference between `useEffect`, `useMemo` and `useCallback`?"
- "How would you fetch data in a React component?"
- "What's a custom hook? Give an example you've written."

---

## 12. Going deeper

- [react.dev: Synchronizing with Effects](https://react.dev/learn/synchronizing-with-effects)
- [react.dev: You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) —
  essential reading.
- [react.dev: Reusing Logic with Custom Hooks](https://react.dev/learn/reusing-logic-with-custom-hooks)
- [React Compiler documentation](https://react.dev/learn/react-compiler)

**Next:** [Chapter 3 — State Management](03-state-management.md) decides where each piece
of Beacon's state should live.
