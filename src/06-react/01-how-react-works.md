# How React Works

> **🔄 Current (as of October 2026):** This book targets React 19 (19.2 and later), with
> the **React Compiler** (stable since October 2025) for automatic memoization, Vite for
> client-rendered apps, and React Router for routing. Check react.dev for the latest
> minor release.

React is a library for building user interfaces from **components**. Its central idea is
simple and was radical when it appeared: instead of writing code that *changes* the page
step by step ("find this element, set its text, add this class"), you write code that
*describes* what the page should look like for the current data, and React works out how
to update the page to match.

That idea makes UIs far easier to reason about. It also introduces a mental model that's
different from anything in C#, and most React bugs come from fighting it. This chapter
builds that model before we write much React code.

---

## 1. The problem: keeping the UI in sync with data

A ticket list shows tickets. When a ticket is resolved, the status badge changes, the
"open" counter decreases, the ticket moves to another section, and a toast appears. With
**imperative** DOM code, every change is manual:

```js
function onResolved(ticketId) {
  document.querySelector(`#ticket-${ticketId} .badge`).textContent = 'Resolved';
  document.querySelector(`#ticket-${ticketId} .badge`).className = 'badge success';
  const counter = document.querySelector('#open-count');
  counter.textContent = String(Number(counter.textContent) - 1);
  document.querySelector('#resolved-list').append(document.querySelector(`#ticket-${ticketId}`));
  showToast(`Ticket ${ticketId} resolved`);
}
```

Every event handler must know every place in the page that depends on the data. Miss one,
and the UI shows inconsistent state. As features grow, the number of interactions grows
combinatorially. This is the problem that jQuery-era applications drowned in.

---

## 2. The mental model: UI = f(state)

React's answer: **the UI is a function of state.**

```text
   state (data)  ──►  component functions  ──►  description of the UI  ──►  React updates the DOM
        ▲                                                                          │
        └──────────────────────── events change state ◄────────────────────────────┘
```

You write components that, given the current state, **return what the UI should look
like**. When state changes, React calls your components again to get the new description,
compares it with the previous one, and applies the minimal set of DOM changes. You never
touch the DOM directly.

```tsx
function OpenCount({ tickets }: { tickets: TicketSummary[] }) {
  const open = tickets.filter(t => t.status === 'Open' || t.status === 'InProgress').length;
  return <span className="badge">{open} open</span>;
}
```

There's no "decrease the counter" code. The counter is *derived* from the tickets every
time. Resolve a ticket by changing state, and every component that depends on it shows the
right thing automatically.

> **🧱 Durable:** "UI as a function of state" (declarative UI) is now the dominant model
> across platforms: React, Vue, Svelte and Solid on the web; SwiftUI, Jetpack Compose and
> Flutter on mobile; .NET MAUI and Blazor in .NET. Learn it once and it transfers.

---

## 3. Components and JSX

A **component** is a function that returns JSX:

```tsx
type TicketCardProps = {
  ticket: TicketSummary;
  onSelect: (id: TicketId) => void;
};

export function TicketCard({ ticket, onSelect }: TicketCardProps) {
  return (
    <article className="ticket-card" onClick={() => onSelect(ticket.id)}>
      <h3>{ticket.title}</h3>
      <StatusBadge status={ticket.status} />
      {ticket.assignee ? <p>Assigned to {ticket.assignee}</p> : <p className="muted">Unassigned</p>}
    </article>
  );
}
```

### JSX is just function calls

JSX compiles to function calls that create plain objects (**React elements**):

```tsx
<StatusBadge status="Open" />
// compiles to (roughly):
jsx(StatusBadge, { status: 'Open' })
// which returns an object like:
{ type: StatusBadge, props: { status: 'Open' } }
```

So JSX is JavaScript: `{expressions}` embed values, conditionals are `? :` or `&&`, lists
are `.map()`. Attributes follow DOM property names (`className`, `htmlFor`, `onClick`).

### Props: inputs, read-only

**Props** are a component's parameters, passed by the parent. A component must never
modify its props; they belong to the parent. Data flows **down** through props; events flow
**up** through callback props (`onSelect`).

### Composition

Components compose like functions. The `children` prop lets a component wrap arbitrary
content:

```tsx
function Panel({ title, children }: { title: string; children: React.ReactNode }) {
  return (
    <section className="panel">
      <h2>{title}</h2>
      {children}
    </section>
  );
}

<Panel title="My queue">
  <TicketList tickets={tickets} />
</Panel>
```

Composition (not inheritance) is how React UIs are built: the same lesson as Book I,
Chapter 3.

---

## 4. Rendering: what actually happens

### Render and commit

When state changes, React runs two phases:

1. **Render phase**: React calls the affected components (and, by default, their children)
   to produce a new tree of elements. This is just calling functions; nothing touches the
   DOM.
2. **Commit phase**: React compares (**reconciles**) the new tree with the previous one and
   applies the differences to the DOM. Then it runs effects (Chapter 2).

```text
 setState ──► render: call components → new element tree
                 │
                 ▼
              reconcile: diff old tree vs new tree
                 │
                 ▼
              commit: minimal DOM updates (change text, set attribute, insert/remove nodes)
```

"Re-render" therefore means "React called your component function again," not "the DOM
was rebuilt." Re-renders are usually cheap; DOM updates only happen where output changed.

### Rendering must be pure

Because React may call your component many times (and, in development with Strict Mode,
deliberately calls it twice to catch mistakes), the component function must be **pure**:

- Same props and state → same output.
- No side effects during render: no API calls, no subscriptions, no mutating variables
  outside the component, no `Math.random()` or `Date.now()` used for output without care.

Side effects belong in event handlers (things the user did) or effects (synchronizing with
external systems; Chapter 2).

### What triggers a re-render

A component re-renders when:

1. **Its state changes** (`useState`, `useReducer`).
2. **Its parent re-renders** (by default, children re-render with their parent, even if
   their props didn't change).
3. **A context it uses changes** (Chapter 3).

Props changing isn't a separate trigger; props change *because* the parent re-rendered.

---

## 5. Reconciliation and keys

To apply minimal updates, React compares old and new trees with two rules (the *diffing
heuristic*):

1. **Different element types → replace the whole subtree.** If `<TicketCard>` becomes
   `<TicketRow>` at the same position, React unmounts the old component (losing its state)
   and mounts the new one.
2. **Same type → keep the component and update props.** Its state is preserved.

For **lists**, React needs to know which item is which between renders. That's what **keys**
are for:

```tsx
<ul>
  {tickets.map(t => <TicketRow key={t.id} ticket={t} />)}
</ul>
```

With stable keys, inserting a ticket at the top updates one row. Without keys (or with the
array **index** as key), React matches items by position: insert at the top, and every row
receives the next row's props, and **state attached to rows** (an expanded panel, an input's
text) ends up on the wrong ticket.

> **⚠️ What can go wrong:** Using `key={index}` for lists that can be reordered, filtered or
> inserted into causes subtle bugs: inputs showing the wrong values, animations on the
> wrong items. Use a stable, unique ID from the data.

Keys can also deliberately **reset** a component: `<TicketEditor key={ticketId} />` gives a
fresh editor (with fresh state) whenever the ticket changes.

---

## 6. State

**State** is data a component remembers between renders, and changing it triggers a
re-render:

```tsx
function TicketFilters({ onChange }: { onChange: (f: Filters) => void }) {
  const [status, setStatus] = useState<TicketStatus | 'All'>('All');

  return (
    <select
      value={status}
      onChange={e => {
        const next = e.target.value as TicketStatus | 'All';
        setStatus(next);
        onChange({ status: next });
      }}
    >
      <option value="All">All</option>
      {TICKET_STATUSES.map(s => <option key={s} value={s}>{statusMeta[s].label}</option>)}
    </select>
  );
}
```

Key properties:

- **State is a snapshot.** During a render, `status` is a constant. Calling `setStatus`
  doesn't change the variable; it schedules a re-render in which `useState` returns the new
  value.
- **Updates are batched.** Several `setX` calls in one event produce one re-render.
- **State must be treated as immutable.** Replace objects and arrays; don't mutate them:

```tsx
setTickets(ts => ts.map(t => (t.id === id ? { ...t, status: 'Resolved' } : t)));   // ✓ new array, new object
tickets.find(t => t.id === id)!.status = 'Resolved';                               // ✗ mutation: React won't notice
```

React detects changes by **reference equality** (`Object.is`). Mutating an object keeps the
same reference, so React may skip the update. Immutable updates are how React knows
something changed.

### Where state lives

- **Local**: inside the component that uses it.
- **Lifted**: when siblings need the same state, move it to their closest common parent and
  pass it down. "Lifting state up" is the most common React refactoring.
- **Derived values are not state.** If something can be computed from props or other state
  (`openCount`, `filteredTickets`), compute it during render. Storing it separately creates
  two sources of truth that can disagree.

Chapter 3 covers state management at application scale.

---

## 7. Server Components and frameworks (in brief)

React 19 also supports **React Server Components (RSC)**: components that run only on the
server, can read databases directly, and send rendered output (not their code) to the
browser. They're used through **frameworks** such as Next.js and React Router's framework
mode, which also provide server-side rendering (SSR), routing and data loading.

| Approach | Rendering | Good for |
|---|---|---|
| **Client-side SPA** (Vite + React) | In the browser | Authenticated apps behind a login (dashboards, internal tools), where SEO doesn't matter |
| **SSR / static generation** | On the server, then hydrated in the browser | Public content: marketing pages, docs, e-commerce, where first-load speed and SEO matter |
| **Server Components** | On the server, streamed | Data-heavy pages; reducing JavaScript sent to the browser |

Beacon's agent and customer portals are authenticated, interactive applications in front of
an existing ASP.NET Core API. A **client-rendered SPA** is the simpler, appropriate choice;
the public knowledge base could later be server-rendered for SEO (Book VII, Chapter 1
discusses where each piece lives).

> **🧭 When not to use a meta-framework:** If you already have a backend (ASP.NET Core), the
> app sits behind a login, and SEO doesn't matter, adding a Node.js rendering server adds a
> second backend to build, deploy, secure and monitor. A Vite SPA served as static files is
> simpler. Choose SSR/RSC for public, content-heavy or performance-critical first loads.

---

## 8. In practice: Beacon's first screen

Let's build the agent's ticket list with what we have so far: components, props, state,
keys and derived values. Data fetching arrives properly in Chapter 4; for now, a simple
load with the typed client from Book V.

```tsx
// web/src/tickets/StatusBadge.tsx
import type { TicketStatus } from '../api/types';
import { statusMeta } from './statusMeta';

export function StatusBadge({ status }: { status: TicketStatus }) {
  const meta = statusMeta[status];
  return <span className={`badge badge-${meta.tone}`}>{meta.label}</span>;
}
```

```tsx
// web/src/tickets/TicketRow.tsx
import type { TicketSummary } from '../api/types';
import { StatusBadge } from './StatusBadge';

type Props = { ticket: TicketSummary; selected: boolean; onSelect: (id: TicketSummary['id']) => void };

export function TicketRow({ ticket, selected, onSelect }: Props) {
  return (
    <li aria-selected={selected}>
      <button type="button" className="ticket-row" onClick={() => onSelect(ticket.id)}>
        <span className="ticket-id">{ticket.id}</span>
        <span className="ticket-title">{ticket.title}</span>
        <StatusBadge status={ticket.status} />
        <span className={`priority priority-${ticket.priority.toLowerCase()}`}>{ticket.priority}</span>
      </button>
    </li>
  );
}
```

```tsx
// web/src/tickets/TicketQueue.tsx
import { useState } from 'react';
import type { TicketSummary, TicketStatus } from '../api/types';
import { TicketRow } from './TicketRow';

const ACTIVE: readonly TicketStatus[] = ['Open', 'InProgress'];

export function TicketQueue({ tickets }: { tickets: TicketSummary[] }) {
  const [showResolved, setShowResolved] = useState(false);
  const [selectedId, setSelectedId] = useState<TicketSummary['id'] | null>(null);

  // Derived during render: never stored in state
  const visible = showResolved ? tickets : tickets.filter(t => ACTIVE.includes(t.status));
  const activeCount = tickets.filter(t => ACTIVE.includes(t.status)).length;

  return (
    <section aria-labelledby="queue-heading">
      <header>
        <h2 id="queue-heading">My queue <span className="count">{activeCount} active</span></h2>
        <label>
          <input type="checkbox" checked={showResolved} onChange={e => setShowResolved(e.target.checked)} />
          Show resolved
        </label>
      </header>

      {visible.length === 0 ? (
        <p className="empty">Nothing in your queue. 🎉</p>
      ) : (
        <ul className="ticket-list">
          {visible.map(t => (
            <TicketRow key={t.id} ticket={t} selected={t.id === selectedId} onSelect={setSelectedId} />
          ))}
        </ul>
      )}
    </section>
  );
}
```

Notice:

- **`activeCount` and `visible` are derived**, so they can never disagree with `tickets`.
- **Keys are ticket IDs**, so filtering doesn't confuse row identity.
- **Rows are `<button>`s inside list items**, so they're keyboard-accessible and announced
  correctly by screen readers (Chapter 6).
- **`onSelect={setSelectedId}`**: a state setter is a stable function and can be passed
  directly.
- **The component doesn't fetch or mutate**: it receives data and reports intent. That
  makes it trivially testable (Chapter 7) and reusable.

---

## 9. What can go wrong

- **Mutating state or props**, so React doesn't re-render (or renders inconsistently).
- **Side effects during render** (fetching, subscribing, logging analytics on every render).
- **Index keys** in dynamic lists.
- **Duplicated state**: storing derived values, which drift out of sync.
- **Changing component types conditionally** (`isCompact ? <Row/> : <Card/>`) and losing
  child state unexpectedly.
- **Defining components inside other components**: a new function type every render, so
  React remounts it (and its state) every time.
- **Fighting the model**: reaching for `document.querySelector` to change what React owns.

---

## 10. How an experienced engineer thinks about this

- **Describe, don't manipulate.** Think "what should the UI be for this state?"
- **Find the minimal state**, derive everything else.
- **Keep render pure**; put side effects in events and effects.
- **Data down, events up.** Components receive data and report intent.
- **Identity matters**: keys and component types decide what React keeps and what it resets.

---

## 11. Check yourself

**Questions**

1. What does "UI is a function of state" mean, and what problem does it solve?
2. What does JSX compile to?
3. What happens in the render phase vs the commit phase?
4. What triggers a re-render? Does a re-render always change the DOM?
5. Why must render be pure?
6. Why are keys needed, and why is the array index a bad key for dynamic lists?
7. Why is mutating state a bug in React?
8. When would you choose a client-rendered SPA over SSR or Server Components?

**Exercises**

1. Build `TicketQueue` with sample data, then reorder the list with a sort dropdown using
   index keys and an `<input>` in each row. Observe the bug, then fix the keys.
2. Add a "selected ticket" detail panel next to the list, lifting state as needed.
3. Find a place where you'd be tempted to store derived state, and compute it instead.
4. Enable React DevTools' "highlight updates" and watch which components re-render when
   the checkbox toggles.

**Interview-style questions**

- "How does React decide what to update in the DOM?"
- "What are keys for in React?"
- "What's the difference between props and state?"
- "When would you use Server Components or SSR?"

---

## 12. Going deeper

- [react.dev: Thinking in React](https://react.dev/learn/thinking-in-react) — the official
  introduction to the mental model.
- [react.dev: Render and Commit](https://react.dev/learn/render-and-commit) and
  [Preserving and Resetting State](https://react.dev/learn/preserving-and-resetting-state)
- Dan Abramov, [A Complete Guide to useEffect](https://overreacted.io/a-complete-guide-to-useeffect/)
  (preparation for the next chapter).

**Next:** [Chapter 2 — Hooks](02-hooks.md) covers state, effects, refs and custom hooks in
depth.
