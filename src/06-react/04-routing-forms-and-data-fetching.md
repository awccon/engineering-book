# Routing, Forms and Data Fetching

Chapter 3 decided where Beacon's state lives: server data in a data cache, navigation
state in the URL, form state in forms. This chapter builds those three pieces: routing
with React Router, server state with TanStack Query, and forms with validation that
cooperates with the API's problem-details errors. Together they form the backbone of
almost every React application that talks to an API.

> **🔄 Current (as of October 2026):** This chapter uses React Router v7 (the successor to
> both React Router v6 and Remix) in its client-side "data/declarative" modes, TanStack
> Query v5, React Hook Form v7 and Zod. All have stable, widely adopted APIs; check their
> docs for minor changes.

---

## 1. The problem: three hard things every app needs

1. **Routing**: map URLs to screens, keep the URL in sync with what's shown, support
   deep links, Back/Forward, and code splitting per route.
2. **Data fetching**: load server data with caching, deduplication, loading and error
   states, background refresh, invalidation after changes, and no race conditions.
3. **Forms**: track values, validate on the client for fast feedback, submit, show server
   validation errors, prevent double submission, and stay accessible.

Each is easy to do badly by hand, and well-solved by libraries.

---

## 2. Routing

### The mental model

In a **single-page application**, the browser loads one HTML page and JavaScript. When the
user navigates, the router intercepts the link click, updates the URL with the History API
(`history.pushState`), and renders the matching components, without a full page load.

```text
 URL: /tickets/T-42?tab=comments
        │
        ▼  route matching
 <AppLayout>                       path: /
   <TicketsLayout>                 path: tickets
     <TicketDetail ticketId=T-42>  path: :ticketId     search param: tab=comments
```

Routes are **nested**: layouts render shared UI (navigation, sidebar) and an `<Outlet />`
where child routes appear.

### Defining routes

```tsx
// web/src/router.tsx
import { createBrowserRouter, Navigate } from 'react-router';
import { AppLayout } from './layout/AppLayout';
import { RouteError } from './layout/RouteError';

export const router = createBrowserRouter([
  {
    path: '/',
    element: <AppLayout />,
    errorElement: <RouteError />,
    children: [
      { index: true, element: <Navigate to="/tickets" replace /> },
      {
        path: 'tickets',
        lazy: () => import('./tickets/TicketsPage'),            // code-split per route
        children: [
          { path: ':ticketId', lazy: () => import('./tickets/TicketDetailRoute') },
        ],
      },
      { path: 'kb', lazy: () => import('./kb/KnowledgeBasePage') },
      { path: '*', element: <NotFound /> },
    ],
  },
]);
```

```tsx
// web/src/main.tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { RouterProvider } from 'react-router';
import { QueryClientProvider } from '@tanstack/react-query';
import { queryClient } from './api/queryClient';
import { router } from './router';

createRoot(document.getElementById('root')!).render(
  <StrictMode>
    <QueryClientProvider client={queryClient}>
      <RouterProvider router={router} />
    </QueryClientProvider>
  </StrictMode>,
);
```

`lazy` routes load their code only when visited, keeping the initial bundle small.

### Reading and writing URL state

```tsx
const { ticketId } = useParams();                       // route params
const [searchParams, setSearchParams] = useSearchParams();   // query string
const navigate = useNavigate();                          // programmatic navigation

<Link to={`/tickets/${ticket.id}`}>Open</Link>
<NavLink to="/kb" className={({ isActive }) => (isActive ? 'active' : '')}>Knowledge base</NavLink>
```

URL parameters are strings controlled by the user: **parse and validate them** like any
other input (Book V, Chapter 5's `asTicketId`).

### Hosting an SPA

Because the router handles paths on the client, the server must return `index.html` for
any path that isn't a static file (otherwise refreshing `/tickets/T-42` returns 404). Book
VIII configures Nginx and Book IX the cloud host for this "SPA fallback."

---

## 3. Server state with TanStack Query

### The mental model

TanStack Query manages a **cache of server data keyed by query keys**:

```text
 ['tickets', 'list', { status: 'Open', q: 'vpn' }]   → Page<TicketSummary>   (fresh / stale / fetching)
 ['tickets', 'detail', 'T-42']                        → TicketDetail
 ['me']                                               → CurrentUser
```

Components *subscribe* to keys. The library handles fetching, caching, deduplicating
simultaneous requests, refetching stale data on window focus or reconnect, retrying
failures, garbage-collecting unused entries, and cancelling outdated requests.

Two timing settings matter most:

- **`staleTime`**: how long data is considered fresh (no refetch). Default 0: always
  revalidate in the background when a new subscriber mounts.
- **`gcTime`**: how long unused data stays in cache (default 5 minutes).

### Queries

```ts
// web/src/api/queryClient.ts
import { QueryClient } from '@tanstack/react-query';
import { ApiError } from './client';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30_000,
      retry: (failureCount, error) =>
        !(error instanceof ApiError && error.status < 500) && failureCount < 2,   // don't retry 4xx
    },
  },
});
```

```ts
// web/src/tickets/queries.ts
import { infiniteQueryOptions, queryOptions } from '@tanstack/react-query';
import { ticketsApi, type ListTicketsParams } from '../api/client';

export const ticketKeys = {
  all: ['tickets'] as const,
  lists: () => [...ticketKeys.all, 'list'] as const,
  list: (params: ListTicketsParams) => [...ticketKeys.lists(), params] as const,
  detail: (id: string) => [...ticketKeys.all, 'detail', id] as const,
};

export const ticketListQuery = (params: ListTicketsParams) =>
  infiniteQueryOptions({
    queryKey: ticketKeys.list(params),
    queryFn: ({ pageParam, signal }) => ticketsApi.list({ ...params, cursor: pageParam }, signal),
    initialPageParam: undefined as string | undefined,
    getNextPageParam: page => page.nextCursor ?? undefined,     // Book III's cursor pagination
  });

export const ticketDetailQuery = (id: string) =>
  queryOptions({
    queryKey: ticketKeys.detail(id),
    queryFn: ({ signal }) => ticketsApi.get(id, signal),
  });
```

A **query key factory** (`ticketKeys`) keeps keys consistent, which matters for
invalidation: `invalidateQueries({ queryKey: ticketKeys.lists() })` refreshes every ticket
list regardless of filters.

The `signal` passed to `queryFn` is cancelled automatically when the query becomes
irrelevant (the key changed, the component unmounted), which solves the out-of-order
search race condition from Book V, Chapter 2.

```tsx
const { data, status, error, fetchNextPage, hasNextPage, isFetchingNextPage } =
  useInfiniteQuery(ticketListQuery({ status: 'Open', q: debouncedTerm }));
```

### Mutations and invalidation

```ts
export function useAddComment(ticketId: string) {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (body: string) => ticketsApi.addComment(ticketId, body),
    onSuccess: updated => {
      qc.setQueryData(ticketKeys.detail(ticketId), updated);          // we have the new data: use it
      qc.invalidateQueries({ queryKey: ticketKeys.lists() });         // counts in lists changed
    },
  });
}
```

After a change, either **write the server's response into the cache** (when you have it)
or **invalidate** affected queries so they refetch.

### Optimistic updates

For actions that almost always succeed, update the UI immediately and roll back on error:

```ts
useMutation({
  mutationFn: () => ticketsApi.resolve(id),
  onMutate: async () => {
    await qc.cancelQueries({ queryKey: ticketKeys.detail(id) });
    const previous = qc.getQueryData(ticketKeys.detail(id));
    qc.setQueryData(ticketKeys.detail(id), old => old && { ...old, status: 'Resolved' });
    return { previous };
  },
  onError: (_err, _vars, ctx) => qc.setQueryData(ticketKeys.detail(id), ctx?.previous),
  onSettled: () => qc.invalidateQueries({ queryKey: ticketKeys.all }),
});
```

> **🧭 When not to use optimistic updates:** When failures are common or consequential
> (payments, actions with server-side rules that often reject), or when the server computes
> important parts of the result. Showing success and then reverting confuses users. Use
> them for low-risk, high-frequency actions: starring, toggling, reordering, commenting.

### Live updates into the cache

Chapter 2's `useTicketUpdates` now writes SignalR updates straight into the cache, so every
component showing that ticket updates:

```tsx
const qc = useQueryClient();
useTicketUpdates(ticketId, ticket => {
  qc.setQueryData(ticketKeys.detail(ticketId), ticket);
  qc.invalidateQueries({ queryKey: ticketKeys.lists() });
});
```

---

## 4. Loading states, errors and Suspense

Every data-driven view has at least three states: loading, error and success (plus empty).
Design all of them:

```tsx
function TicketDetailRoute() {
  const ticketId = asTicketId(useParams().ticketId ?? '');
  const { data: ticket, status, error } = useQuery(ticketDetailQuery(ticketId));

  if (status === 'pending') return <DetailSkeleton />;
  if (status === 'error') return <ErrorPanel error={error} />;
  return <TicketDetail ticket={ticket} />;
}
```

- **Skeletons** (placeholder shapes) feel faster than spinners for content areas.
- **Error states** should say what happened and offer a retry; for `404`, a "ticket not found"
  message rather than a generic error.
- **Keep previous data** while refetching new filters (`placeholderData: keepPreviousData`)
  so the list doesn't flash empty on every filter change.

### Suspense and error boundaries

React can also express loading declaratively with **Suspense**: a component "suspends"
while its data loads, and the nearest `<Suspense fallback={...}>` shows a fallback. Errors
propagate to the nearest **error boundary**. TanStack Query supports this with
`useSuspenseQuery`, and React Router's `errorElement` acts as a route-level boundary.

```tsx
<ErrorBoundary fallback={<ErrorPanel />}>
  <Suspense fallback={<DetailSkeleton />}>
    <TicketDetailSuspense ticketId={ticketId} />   {/* data is always defined inside */}
  </Suspense>
</ErrorBoundary>
```

The benefit: components can assume data exists; loading and error handling move to the
layout.

### Avoiding waterfalls

A **request waterfall** happens when nested components each fetch after the parent finishes
rendering: page → list → detail → comments, one after another. Mitigations: fetch in
parallel at the route level (React Router `loader`s can prefetch with
`queryClient.ensureQueryData` before rendering), prefetch on hover
(`queryClient.prefetchQuery`), and design APIs that return what a screen needs (Book III,
Chapter 4).

---

## 5. Forms

### Controlled vs uncontrolled inputs

- **Controlled**: React state holds the value; the input displays it and reports changes
  (`value` + `onChange`). Full control, re-renders on every keystroke.
- **Uncontrolled**: the DOM holds the value; you read it on submit (`FormData`, refs).
  Fewer re-renders, less code.

Form libraries like **React Hook Form** use uncontrolled inputs with refs under the hood for
performance, plus a clean API for validation and errors. React 19's **form actions**
(`<form action={fn}>`, `useActionState`, `useFormStatus`) are a built-in, lighter-weight
option, especially in frameworks with server actions.

### Validation: client and server

Client-side validation is for **fast feedback**. Server-side validation is for
**correctness** (Book III, Chapter 5); the client can't be trusted. A good form:

1. Validates on the client with the same rules the server enforces (where practical).
2. Submits.
3. Maps server validation errors (problem details) back to fields.
4. Shows non-field errors (409 conflict, 403, network failures) at the form level.

---

## 6. In practice: Beacon's ticket screens

### The tickets page: URL-driven filters and infinite list

```tsx
// web/src/tickets/TicketsPage.tsx
import { useInfiniteQuery, keepPreviousData } from '@tanstack/react-query';
import { Outlet, useSearchParams } from 'react-router';
import { TICKET_STATUSES, type TicketStatus } from '../api/types';
import { useDebouncedValue } from '../hooks/useDebouncedValue';
import { ticketListQuery } from './queries';
import { TicketQueue } from './TicketQueue';
import { TicketSearch } from './TicketSearch';

const isStatus = (v: string | null): v is TicketStatus => (TICKET_STATUSES as readonly string[]).includes(v ?? '');

export function Component() {     // React Router lazy routes export `Component`
  const [params, setParams] = useSearchParams();
  const status = isStatus(params.get('status')) ? (params.get('status') as TicketStatus) : undefined;
  const term = params.get('q') ?? '';
  const debouncedTerm = useDebouncedValue(term.trim(), 300);

  const query = useInfiniteQuery({
    ...ticketListQuery({ status, q: debouncedTerm || undefined, limit: 25 }),
    placeholderData: keepPreviousData,
  });

  const tickets = query.data?.pages.flatMap(p => p.items) ?? [];

  function update(key: string, value: string | undefined) {
    setParams(prev => {
      const next = new URLSearchParams(prev);
      if (value) next.set(key, value); else next.delete(key);
      return next;
    }, { replace: key === 'q' });           // don't flood history while typing
  }

  return (
    <div className="tickets-layout">
      <div className="tickets-list">
        <TicketSearch value={term} onChange={v => update('q', v)} />
        <StatusFilter value={status} onChange={v => update('status', v)} />

        {query.status === 'pending' ? <ListSkeleton /> :
         query.status === 'error' ? <ErrorPanel error={query.error} onRetry={() => query.refetch()} /> :
         <TicketQueue tickets={tickets} />}

        {query.hasNextPage && (
          <button onClick={() => query.fetchNextPage()} disabled={query.isFetchingNextPage}>
            {query.isFetchingNextPage ? 'Loading…' : 'Load more'}
          </button>
        )}
      </div>
      <Outlet />   {/* ticket detail route renders here */}
    </div>
  );
}
```

Note: search term and status live in the URL, so a link to `/tickets?status=Open&q=vpn`
reproduces the view exactly. (The `q` filter needs a small addition to Book III's list
endpoint, which is exercise 1.)

### The new-ticket form

```bash
pnpm add react-hook-form @hookform/resolvers zod
```

```tsx
// web/src/tickets/NewTicketForm.tsx
import { zodResolver } from '@hookform/resolvers/zod';
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { useForm } from 'react-hook-form';
import { useNavigate } from 'react-router';
import { z } from 'zod';
import { ticketsApi } from '../api/client';
import { TICKET_PRIORITIES } from '../api/types';
import { toFieldErrors } from '../forms/serverErrors';
import { ticketKeys } from './queries';

const schema = z
  .object({
    title: z.string().trim().min(1, 'Title is required').max(200, 'Keep the title under 200 characters'),
    priority: z.enum(TICKET_PRIORITIES),
    description: z.string().max(10_000).optional(),
  })
  .refine(v => v.priority !== 'Urgent' || (v.description?.trim().length ?? 0) > 0, {
    path: ['description'],
    message: 'Urgent tickets need a description so on-call can act on them.',   // same rule as the server (Book III, Ch. 5)
  });

type Values = z.infer<typeof schema>;

export function NewTicketForm() {
  const qc = useQueryClient();
  const navigate = useNavigate();
  const { register, handleSubmit, setError, formState: { errors, isSubmitting } } =
    useForm<Values>({ resolver: zodResolver(schema), defaultValues: { priority: 'Normal' } });

  const create = useMutation({ mutationFn: ticketsApi.create });

  const onSubmit = handleSubmit(async values => {
    try {
      const ticket = await create.mutateAsync(values);
      qc.setQueryData(ticketKeys.detail(ticket.id), ticket);
      await qc.invalidateQueries({ queryKey: ticketKeys.lists() });
      navigate(`/tickets/${ticket.id}`);
    } catch (err) {
      const { fieldErrors, formError } = toFieldErrors<Values>(err, ['title', 'priority', 'description']);
      for (const [field, message] of Object.entries(fieldErrors)) setError(field as keyof Values, { message });
      if (formError) setError('root', { message: formError });
    }
  });

  return (
    <form onSubmit={onSubmit} noValidate aria-describedby={errors.root ? 'form-error' : undefined}>
      {errors.root && <p id="form-error" role="alert" className="form-error">{errors.root.message}</p>}

      <label htmlFor="title">Title</label>
      <input id="title" {...register('title')} aria-invalid={!!errors.title} aria-describedby="title-error" />
      {errors.title && <p id="title-error" className="field-error">{errors.title.message}</p>}

      <label htmlFor="priority">Priority</label>
      <select id="priority" {...register('priority')}>
        {TICKET_PRIORITIES.map(p => <option key={p} value={p}>{p}</option>)}
      </select>

      <label htmlFor="description">Description</label>
      <textarea id="description" rows={6} {...register('description')}
                aria-invalid={!!errors.description} aria-describedby="description-error" />
      {errors.description && <p id="description-error" className="field-error">{errors.description.message}</p>}

      <button type="submit" disabled={isSubmitting}>{isSubmitting ? 'Creating…' : 'Create ticket'}</button>
    </form>
  );
}
```

What this form gets right:

- **Same rules as the server**, including the conditional "urgent needs description" rule,
  for instant feedback. The server still enforces them.
- **Server errors land on the right fields** via `toFieldErrors` (Book V, Chapter 5);
  anything else shows at the form level.
- **Double submission prevented** with `isSubmitting`, and duplicates are impossible anyway
  thanks to the idempotency key in `ticketsApi.create`.
- **Accessible errors**: `aria-invalid`, `aria-describedby` and `role="alert"` (Chapter 6).
- **Cache updated**: the detail cache is seeded with the response, lists invalidated, then
  navigation shows the new ticket instantly.

---

## 7. What can go wrong

- **Fetching in effects** without a cache: races, duplicate requests, stale data.
- **Inconsistent query keys**, so invalidation misses some caches.
- **Forgetting to invalidate** after mutations: stale lists.
- **Request waterfalls** from nested fetching components.
- **Unvalidated URL params** used directly in API calls.
- **Missing SPA fallback** on the server: 404 on refresh.
- **Client-only validation**, or client rules that drift from server rules.
- **Swallowed server errors**: a generic "Something went wrong" when the API said exactly
  which field was invalid.
- **Optimistic updates for risky actions.**

---

## 8. How an experienced engineer thinks about this

- **The URL is the app's primary state**: design routes and search params deliberately.
- **Server state is a cache**: keys, staleness and invalidation are the design decisions.
- **Every view has loading, error, empty and success states**, designed up front.
- **Validate twice**: client for experience, server for truth, with the same rules where
  possible.
- **Avoid waterfalls**: fetch early, in parallel, at the route level.

---

## 9. Check yourself

**Questions**

1. How does client-side routing work without full page loads? What must the server do?
2. What are query keys, and why use a key factory?
3. What's the difference between `staleTime` and `gcTime`?
4. When do you update the cache directly vs invalidate?
5. When are optimistic updates appropriate?
6. What's a request waterfall, and how do you avoid one?
7. Why validate on both the client and the server? How do server errors reach form fields?

**Exercises**

1. Add a `q` (search) parameter to Beacon.Api's list endpoint and wire it to the URL-driven
   search box.
2. Prefetch ticket details on hover with `queryClient.prefetchQuery`.
3. Implement an optimistic "assign to me" action with rollback, and test the rollback by
   making the API fail.
4. Rewrite the new-ticket form with React 19's `useActionState` and compare the two
   approaches.

**Interview-style questions**

- "How do you handle data fetching in React?"
- "How do you keep client and server validation consistent?"
- "What's the difference between controlled and uncontrolled inputs?"
- "How do you handle pagination in a React app?"

---

## 10. Going deeper

- [React Router documentation](https://reactrouter.com/)
- [TanStack Query documentation](https://tanstack.com/query/latest) and TkDodo's blog
  (tkdodo.eu).
- [React Hook Form documentation](https://react-hook-form.com/)
- [react.dev: `useActionState`](https://react.dev/reference/react/useActionState)

**Next:** [Chapter 5 — Authentication in the Browser](05-authentication-in-the-browser.md)
signs users in, securely.
