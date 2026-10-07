# State Management

"Which state management library should we use?" is one of the most debated questions in
frontend development, and usually the wrong first question. Most state problems in React
apps don't come from the choice of library; they come from **putting state in the wrong
place**: server data copied into a global store and going stale, form drafts in global
state, filters that should be in the URL, and everything in one giant context that
re-renders the whole app on every keystroke.

This chapter classifies the kinds of state an application has, shows where each belongs,
and covers context and external stores for the cases that genuinely need them.

---

## 1. The problem: state everywhere

Beacon's agent screen involves:

- the list of tickets (from the server),
- the selected ticket and its comments (from the server),
- the current filters and sort (chosen by the user),
- whether the "new ticket" dialog is open,
- the text being typed in a comment box,
- the logged-in user and their permissions,
- the theme and density preferences,
- a "live" indicator for the SignalR connection.

Each has a different **owner**, **lifetime**, **sharing scope** and **source of truth**.
Treating them all the same is how state management becomes painful.

---

## 2. The mental model: kinds of state

| Kind | Examples | Source of truth | Where it belongs |
|---|---|---|---|
| **Server state** | Tickets, comments, users | The server (a remote cache on the client) | A **data-fetching cache** (TanStack Query, framework loaders) |
| **URL state** | Filters, sort, page cursor, selected ticket ID, active tab | The URL | The **router** / search params |
| **Local UI state** | Dialog open, input text, hover, expanded rows | The component | **`useState`/`useReducer`** in the component |
| **Form state** | Field values, touched, validation errors | The form | A form hook or library (Chapter 4) |
| **Global client state** | Current user session, theme, feature flags, toasts | The client | **Context** or a small **store** |
| **Derived state** | Counts, filtered lists, "can edit" | Computed from the above | **Computed during render** (never stored) |

> **🧱 Durable:** For each piece of state, ask: *who owns it, how long does it live, who
> needs it, and where is its source of truth?* The answers usually make the placement
> obvious, and very little ends up "global."

### Server state is a cache, not state

The biggest shift in modern React architecture: data from the server isn't *your* state.
It's a **cached copy** of someone else's state, which can be stale the moment it arrives
(another agent resolved the ticket). It needs caching, deduplication, background
refetching, invalidation after mutations, and loading/error states. That's a solved problem
with dedicated libraries; Chapter 4 uses TanStack Query. Copying server data into Redux or
`useState` means reimplementing all of it, usually badly.

### The URL is state

If a user copies the link, presses Back, or refreshes, should they see the same thing? If
yes, the state belongs in the URL:

```text
/tickets?status=Open&assignee=me&sort=-priority        filters
/tickets/T-42                                          selected ticket
/tickets/T-42?tab=history                              active tab
```

URL state is shareable, bookmarkable, survives refresh, and works with the Back button for
free. Chapter 4 covers routing.

---

## 3. Local state and lifting

Start every piece of state as **local** as possible. Lift it only when another component
needs it, to the **closest common ancestor**:

```text
 TicketsPage            ← owns selectedId (both children need it)
 ├─ TicketQueue          receives selectedId, onSelect
 └─ TicketDetailPanel    receives selectedId
```

Signs state is too high: a keystroke in one input re-renders the whole page; components
receive props they only pass through (**prop drilling** through many layers).

Signs state is too low: two components show inconsistent copies of the same thing; effects
copy state between siblings.

### Prop drilling is not always a problem

Passing props through two or three layers is explicit and easy to follow. It becomes a
problem when many intermediate components carry props they don't use. Before reaching for
context, try **composition**: pass components as `children` so the data flows directly to
where it's used:

```tsx
// Instead of Layout → Sidebar → UserMenu → Avatar all taking `user`:
<Layout sidebar={<Sidebar><UserMenu user={user} /></Sidebar>}>
  <TicketsPage />
</Layout>
```

---

## 4. Context

**Context** passes a value to any descendant without props:

```tsx
type Session = { user: CurrentUser; permissions: ReadonlySet<Permission> };

const SessionContext = createContext<Session | null>(null);

export function SessionProvider({ session, children }: { session: Session; children: ReactNode }) {
  return <SessionContext value={session}>{children}</SessionContext>;     // React 19: <Context> as provider
}

export function useSession(): Session {
  const session = useContext(SessionContext);
  if (!session) throw new Error('useSession must be used within <SessionProvider>');
  return session;
}
```

The custom hook with a clear error message is a good pattern: consumers can't forget the
provider, and the type is non-nullable.

### Context's performance characteristic

When a context's value changes, **every component that reads it re-renders**. That's fine
for values that change rarely (session, theme, locale, feature flags). It's a problem for
values that change often (form fields, live data, a big object where one field updates
frequently).

Mitigations:

- **Split contexts** by update frequency: `SessionContext` separate from `ToastContext`.
- **Keep provider values stable**: don't create a new object every render
  (`value={{ user, logout }}` creates a new object each time; memoize it or let the React
  Compiler do it).
- For frequently changing shared state, use an **external store** with selective
  subscriptions (next section).

> **🧭 When not to use context:** Context is a dependency-injection mechanism for React
> trees, not a state management solution by itself. Don't use it for server data (use the
> data cache), for URL state (use the router), or for high-frequency updates shared widely
> (use a store with selectors).

---

## 5. External stores

When you genuinely have **client-side state shared across distant components that changes
often**, a small external store lets components subscribe to just the slice they need:

| Library | Style | Notes |
|---|---|---|
| **Zustand** | Minimal hook-based store | Small API, selectors, no providers needed; very popular |
| **Redux Toolkit** | Centralized store, reducers, actions | Structured, great devtools, more ceremony; strong in large legacy codebases |
| **Jotai** | Atoms (small independent pieces of state) | Fine-grained, composable |
| **XState** | State machines and statecharts | Complex workflows with many explicit states |

Example with Zustand:

```ts
import { create } from 'zustand';

type UiState = {
  density: 'comfortable' | 'compact';
  liveStatus: 'connecting' | 'live' | 'offline';
  setDensity: (d: UiState['density']) => void;
  setLiveStatus: (s: UiState['liveStatus']) => void;
};

export const useUiStore = create<UiState>()(set => ({
  density: 'comfortable',
  liveStatus: 'connecting',
  setDensity: density => set({ density }),
  setLiveStatus: liveStatus => set({ liveStatus }),
}));

// A component re-renders only when the selected slice changes:
const liveStatus = useUiStore(s => s.liveStatus);
```

Under the hood these libraries use `useSyncExternalStore`, React's official API for
subscribing to state that lives outside React.

### Do you need one?

In modern React apps, once server state moves to a data-fetching cache and URL state moves
to the router, the remaining global client state is usually small: session, preferences,
a few UI flags. **Context plus local state often covers it.** Add a store when profiling or
code complexity shows you need one, not by default.

> **🧭 When not to use Redux (or any global store):** For server data (the most common
> misuse), for state used by one component or one subtree, and for form state. A global
> store full of API responses, loading flags and form drafts is the classic "state
> management pain" that drove the community toward specialized tools.

---

## 6. Immutability and update patterns

Whatever the container, React state must be updated immutably (Chapter 1). Nested updates
get verbose:

```ts
setState(s => ({
  ...s,
  tickets: s.tickets.map(t =>
    t.id === id ? { ...t, comments: [...t.comments, newComment] } : t,
  ),
}));
```

Options:

- **Flatten** state (store entities by ID, not deeply nested).
- **Immer** lets you write "mutating" code against a draft that produces an immutable
  update (Redux Toolkit uses it internally):

```ts
import { produce } from 'immer';
setState(produce(draft => {
  draft.tickets.find(t => t.id === id)?.comments.push(newComment);
}));
```

- Often the best option: **let the data cache own server data**, so you rarely hand-update
  nested server objects at all.

---

## 7. In practice: Beacon's state map

Applying the classification to Beacon's agent app:

| State | Kind | Home |
|---|---|---|
| Ticket lists, ticket details, comments | Server | TanStack Query cache (Chapter 4), updated live via SignalR events |
| Filters, sort, search term, cursor | URL | Search params: `/tickets?status=Open&q=vpn` |
| Selected ticket | URL | Route param: `/tickets/:ticketId` |
| Comment draft, dialog open, expanded sections | Local | `useState` in the component |
| New-ticket form | Form | Form hook with Zod validation (Chapter 4) |
| Current user, permissions | Global, rarely changes | `SessionContext` (loaded once from `/api/me` via the BFF; Chapter 5) |
| Density preference | Global, rarely changes | Small Zustand store persisted to `localStorage` |
| Live connection status | Global, changes occasionally | Same Zustand store (set by the SignalR connection) |
| Open count, "can resolve", filtered views | Derived | Computed during render |

Permission checks become a small derived helper over the session context:

```tsx
// web/src/session/permissions.ts
export type Permission = 'tickets.resolve' | 'tickets.reopen' | 'tickets.assign' | 'kb.publish';

export function useCan(permission: Permission): boolean {
  return useSession().permissions.has(permission);
}

// usage
const canResolve = useCan('tickets.resolve');
{canResolve && <button onClick={resolve}>Resolve</button>}
```

> **⚠️ What can go wrong:** Hiding the button is a **UX** decision, not a security control.
> The API enforces authorization on every request (Book III, Chapter 6). Never treat
> client-side permission checks as protection.

The connection status store integrates with the SignalR connection from Chapter 2:

```ts
// web/src/realtime/connection.ts (excerpt)
hub.onreconnecting(() => useUiStore.getState().setLiveStatus('connecting'));
hub.onreconnected(() => useUiStore.getState().setLiveStatus('live'));
hub.onclose(() => useUiStore.getState().setLiveStatus('offline'));
```

```tsx
// web/src/layout/LiveIndicator.tsx
export function LiveIndicator() {
  const status = useUiStore(s => s.liveStatus);   // re-renders only when this slice changes
  return (
    <span className={`live live-${status}`} role="status" aria-live="polite">
      {status === 'live' ? 'Live' : status === 'connecting' ? 'Reconnecting…' : 'Offline: updates paused'}
    </span>
  );
}
```

`getState()` lets non-React code (the SignalR callbacks) update the store, which is one of
the main reasons to choose an external store over context for this piece of state.

The resulting architecture has **no global store of server data at all**. That's the most
important decision in this chapter.

---

## 8. What can go wrong

- **Server data in global client state**, going stale and duplicating cache logic.
- **URL-worthy state in component state**: filters lost on refresh, links that can't be
  shared, broken Back button.
- **Derived state stored separately**, drifting out of sync.
- **One giant context** re-rendering everything on every change.
- **Unstable context values** (new objects every render).
- **Premature global stores** for state used in one place.
- **Mutating state** inside stores or reducers (without Immer).
- **Client-side permission checks treated as security.**

---

## 9. How an experienced engineer thinks about this

- **Classify first, then place.** Server, URL, local, form, global, derived.
- **Keep state as local as possible**, lifted only as far as needed.
- **The server owns server data**; the client caches it.
- **The URL is the best global state store** for anything navigational.
- **Add a store when you can name the problem it solves.**

---

## 10. Check yourself

**Questions**

1. What are the six kinds of state in section 2? Give a Beacon example of each.
2. Why should server data live in a data-fetching cache rather than a global store?
3. Which state belongs in the URL, and what do you gain?
4. What's prop drilling, and how can composition reduce it?
5. What's the performance characteristic of context, and how do you mitigate it?
6. When is an external store like Zustand or Redux justified?
7. Why are client-side permission checks not a security control?

**Exercises**

1. Audit a React app you know: classify each piece of state and note anything in the wrong
   place.
2. Move Beacon's ticket filters from component state into URL search params (preview of
   Chapter 4).
3. Build `SessionContext` with a `useSession` hook that throws a helpful error outside the
   provider, and test it.
4. Persist the density preference with Zustand's `persist` middleware, validating the
   stored value on load.

**Interview-style questions**

- "How do you decide where state should live in a React application?"
- "When would you use Context vs Redux vs Zustand?"
- "What's the difference between server state and client state?"

---

## 11. Going deeper

- [react.dev: Managing State](https://react.dev/learn/managing-state)
- Kent C. Dodds, [Application State Management with React](https://kentcdodds.com/blog/application-state-management-with-react)
- TkDodo (Dominik Dorfmeister), [Practical React Query](https://tkdodo.eu/blog/practical-react-query) series
- [Zustand documentation](https://zustand.docs.pmnd.rs/)

**Next:** [Chapter 4 — Routing, Forms and Data Fetching](04-routing-forms-and-data-fetching.md)
puts URL state, server state and form state into practice.
