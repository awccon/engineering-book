# Testing and Frontend Architecture

A frontend starts small: a few components in a `components/` folder. Two years later it's
800 files, every feature touches every folder, a "shared" `utils.ts` has 3,000 lines, and
nobody dares change the `Button` because forty screens depend on its quirks. Tests either
don't exist or break whenever markup changes.

This chapter covers both sides of keeping a frontend healthy as it grows: **tests** that
give confidence without breaking on every refactor, and an **architecture** that keeps
features independent.

---

## 1. The problem: UIs change constantly

Frontend code changes faster than almost any other code: design tweaks, copy changes, new
layouts, component library upgrades. Tests and structure must survive that churn:

- Tests coupled to markup details (CSS classes, DOM structure, component internals) break
  on every visual change, and get deleted.
- A structure organized by technical type (`components/`, `hooks/`, `utils/`) scatters each
  feature across the codebase, so every change touches many folders and ownership is unclear.

---

## 2. Testing: the mental model

Book I, Chapter 14's principles apply directly: **test behavior, not implementation**, and
choose the test level by where bugs live.

| Level | Tests | Tools | Use for |
|---|---|---|---|
| **Unit** | Pure functions, reducers, hooks with logic | Vitest | Business rules in the client (formatting, validation, reducers, permission helpers) |
| **Component / integration** | Components rendered with their children, interacting like a user | Vitest + **React Testing Library** + **MSW** | Most UI behavior: forms, lists, states, error handling |
| **End-to-end** | The real app in a real browser against a real (or realistic) backend | **Playwright** (Book VII, Chapter 3) | Critical journeys: sign in, create a ticket, resolve it |
| **Visual regression** | Screenshots compared over time | Playwright screenshots, Chromatic | Design systems, layout-sensitive components |

For frontends, the **testing trophy** shape fits well: a broad middle of component and
integration tests, because most frontend bugs are in how pieces fit together (state +
rendering + API responses), not in isolated functions.

> **🧱 Durable:** *"The more your tests resemble the way your software is used, the more
> confidence they can give you."* (Kent C. Dodds) Users don't know about props, state or
> CSS classes. They see text, labels and roles, and they click, type and wait.

---

## 3. React Testing Library

React Testing Library (RTL) renders components into a simulated DOM (jsdom or happy-dom)
and gives you queries that find elements **the way users and assistive technology do**:

| Query (in order of preference) | Finds by |
|---|---|
| `getByRole('button', { name: 'Resolve' })` | Accessible role and name |
| `getByLabelText('Title')` | Form label |
| `getByPlaceholderText`, `getByText` | Visible text |
| `getByDisplayValue` | Current input value |
| `getByAltText`, `getByTitle` | Image alt, title |
| `getByTestId('...')` | `data-testid`: last resort |

Role-based queries have a valuable side effect: if a test can't find your button by role
and name, neither can a screen reader (Chapter 6). Tests push you toward accessible markup.

```tsx
// web/src/tickets/TicketQueue.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, expect, it, vi } from 'vitest';
import { TicketQueue } from './TicketQueue';
import { aTicket } from '../test/builders';

describe('TicketQueue', () => {
  it('hides resolved tickets until "Show resolved" is checked', async () => {
    const user = userEvent.setup();
    render(<TicketQueue tickets={[aTicket({ title: 'VPN drops' }), aTicket({ title: 'Old typo', status: 'Resolved' })]} />);

    expect(screen.getByRole('button', { name: /VPN drops/ })).toBeInTheDocument();
    expect(screen.queryByRole('button', { name: /Old typo/ })).not.toBeInTheDocument();

    await user.click(screen.getByRole('checkbox', { name: 'Show resolved' }));

    expect(screen.getByRole('button', { name: /Old typo/ })).toBeInTheDocument();
  });

  it('shows the active count', () => {
    render(<TicketQueue tickets={[aTicket(), aTicket({ status: 'InProgress' }), aTicket({ status: 'Closed' })]} />);
    expect(screen.getByRole('heading', { name: /My queue/ })).toHaveTextContent('2 active');
  });
});
```

Notes:

- **`userEvent`** simulates real interactions (focus, keyboard, pointer events) rather than
  firing synthetic events directly.
- **`aTicket(...)`** is a test data builder, like Book I, Chapter 14's `TicketBuilder`.
- **`find*` queries** wait for elements to appear (async UI); `query*` returns null instead of
  throwing, for asserting absence.
- No test inspects state, props or CSS classes. Rename the state variable, restructure the
  markup, change the styling: the tests still pass if the behavior is the same.

---

## 4. Mocking the network with MSW

Components that fetch data need an API in tests. Mocking `fetch` or the API module ties
tests to implementation details. **Mock Service Worker (MSW)** intercepts requests at the
network level and returns responses you define, so the real API client, TanStack Query
and components all run as in production:

```ts
// web/src/test/handlers.ts
import { http, HttpResponse } from 'msw';
import { aTicket } from './builders';

export const handlers = [
  http.get('/api/tickets', () =>
    HttpResponse.json({ items: [aTicket({ id: 'T-1', title: 'VPN drops' })], nextCursor: null })),

  http.post('/api/tickets', async ({ request }) => {
    const body = (await request.json()) as { title: string };
    if (body.title === 'duplicate')
      return HttpResponse.json(
        { title: 'Validation failed', status: 400, errors: { Title: ['A ticket with this title is already open.'] } },
        { status: 400, headers: { 'Content-Type': 'application/problem+json' } });
    return HttpResponse.json(aTicket({ id: 'T-99', title: body.title }), { status: 201 });
  }),
];
```

```ts
// web/src/test/setup.ts
import '@testing-library/jest-dom/vitest';
import { afterAll, afterEach, beforeAll } from 'vitest';
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

A form test then exercises the full client-side path, including server validation errors
mapped onto fields (Chapter 4):

```tsx
it('shows the server validation error next to the title field', async () => {
  const user = userEvent.setup();
  renderWithProviders(<NewTicketForm />);

  await user.type(screen.getByLabelText('Title'), 'duplicate');
  await user.click(screen.getByRole('button', { name: 'Create ticket' }));

  expect(await screen.findByText('A ticket with this title is already open.')).toBeInTheDocument();
  expect(screen.getByLabelText('Title')).toHaveAttribute('aria-invalid', 'true');
});
```

MSW handlers are reusable in the browser too (for local development without a backend, or
demos), and can be generated from the OpenAPI document (Book VII, Chapter 2) so mocks stay
in sync with the real API.

---

## 5. What to test in a frontend

**Test thoroughly:**

- User-visible behavior of each feature: the happy path, empty states, loading and error
  states.
- Forms: client validation, server error mapping, double-submit prevention.
- Permission-dependent UI (what each role sees).
- Pure logic: reducers, formatters, mappers, URL parsing.
- Regressions for every bug.

**Test lightly:**

- Pure presentational components with no logic (a snapshot or visual test, or nothing).
- Third-party library behavior.
- Exact markup and styling (use visual regression for design-critical components instead).

> **⚠️ What can go wrong:** Large **snapshot tests** of component trees seem convenient, but
> they fail on every markup change, reviewers approve updates without reading them, and they
> end up asserting nothing. Prefer targeted assertions about behavior.

---

## 6. Frontend architecture: the mental model

### Organize by feature

```text
web/src/
  app/                     # composition root: providers, router, layout
  features/
    tickets/
      api/                 # queries, mutations, keys for this feature
      components/          # TicketQueue, TicketRow, StatusBadge...
      routes/              # route components: TicketsPage, TicketDetailRoute
      model/               # types, reducers, pure logic (statusMeta, editorReducer)
      index.ts             # public API of the feature
    knowledge-base/
    admin/
  shared/
    ui/                    # design-system components: Button, Dialog, Field, Badge
    api/                   # HTTP client, ApiError, generated types
    lib/                   # truly generic utilities (useDebouncedValue, formatters)
  session/                 # auth/session context, permissions
  test/                    # builders, MSW handlers, render helpers
```

**Feature folders** (sometimes called *vertical slices*) keep everything a feature needs
together. A change to ticket comments touches `features/tickets/` and nothing else. It's
the frontend version of Book XIII's *modular monolith*.

### Dependency rules

```text
 app ──► features ──► shared
           │  ✗ feature → feature (directly)
           └─► session
```

- **Features don't import from each other's internals.** If the knowledge base needs to show
  a ticket badge, either it uses the ticket feature's **public API** (`features/tickets/index.ts`)
  or the badge moves to `shared/ui`.
- **`shared` never imports from features.**
- Enforce with lint rules (`eslint-plugin-boundaries`, `import/no-restricted-paths`), the
  same way Book I, Chapter 14 enforced backend layering with an architecture test.

### Layers within a component tree

A useful separation of responsibilities (not strict layers):

- **Route components**: read URL params, start queries, handle loading/error states, compose
  the page.
- **Feature components**: feature-specific UI, receive data via props, raise events.
- **Shared UI components**: generic, accessible, no business knowledge (a `Dialog` doesn't
  know what a ticket is).

### Design systems

As an app grows, consistent UI requires a **design system**: tokens (colors, spacing,
typography), and accessible base components. Options: build on a headless library (React
Aria, Radix) with your own styling, use a complete component library (MUI, Mantine,
Fluent UI), or a copy-in collection (shadcn/ui). Wrap third-party components in your own
`shared/ui` components, so replacing the library later touches one folder.

> **🧭 When not to over-architect:** A small app with a handful of screens doesn't need
> feature folders, boundary lint rules and a design system. Start simple, and introduce
> structure when you feel the pain of a growing codebase: features stepping on each other,
> unclear ownership, slow onboarding.

---

## 7. In practice: Beacon's frontend structure and test suite

### Restructure into features

Move Beacon's code into the structure above. The ticket feature's public API:

```ts
// web/src/features/tickets/index.ts
export { StatusBadge } from './components/StatusBadge';
export { ticketKeys, ticketDetailQuery } from './api/queries';
export type { TicketSummary, TicketStatus, TicketPriority } from '../../shared/api/types';
```

The knowledge base's "related tickets" panel imports only from `features/tickets` (the
index), never from `features/tickets/components/...`. An ESLint boundary rule enforces it.

### A render helper with providers

```tsx
// web/src/test/render.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { render } from '@testing-library/react';
import { MemoryRouter } from 'react-router';
import { SessionProvider } from '../session/SessionContext';
import { aSession } from './builders';

export function renderWithProviders(
  ui: React.ReactElement,
  { route = '/', session = aSession() } = {},
) {
  const queryClient = new QueryClient({ defaultOptions: { queries: { retry: false } } });   // fresh cache per test
  return render(
    <QueryClientProvider client={queryClient}>
      <SessionProvider session={session}>
        <MemoryRouter initialEntries={[route]}>{ui}</MemoryRouter>
      </SessionProvider>
    </QueryClientProvider>,
  );
}
```

A fresh `QueryClient` per test prevents cached data leaking between tests (the frontend
equivalent of test isolation in Book I, Chapter 14), and disabling retries makes error-state
tests fast.

### Permission-dependent UI, tested per role

```tsx
it.each([
  ['agent', true],
  ['customer', false],
])('a %s %s see the Resolve button', async (role, canSee) => {
  renderWithProviders(<TicketDetailRoute />, { route: '/tickets/T-1', session: aSession({ roles: [role] }) });
  await screen.findByRole('heading', { name: /VPN drops/ });
  expect(!!screen.queryByRole('button', { name: 'Resolve' })).toBe(canSee);
});
```

Remember: this tests the *UX*. The API's authorization tests (Book III, Chapter 6) test the
*security*.

### Running it

```json
// web/package.json (scripts)
"test": "vitest",
"test:ci": "vitest run --coverage"
```

```ts
// web/vite.config.ts (test section)
test: {
  environment: 'jsdom',
  setupFiles: ['./src/test/setup.ts'],
  css: false,
},
```

CI runs the suite on every pull request alongside the .NET tests (Book V, Chapter 3).

---

## 8. What can go wrong

- **Tests coupled to implementation**: asserting state, props, class names or DOM structure.
- **Mocking modules** instead of the network, so the real data layer is never tested.
- **Shared query cache between tests**, causing order-dependent failures.
- **Snapshot tests nobody reads.**
- **Only E2E tests**: slow, flaky, and hard to diagnose; or **no E2E tests**: integration of
  the real pieces never verified.
- **Folder-by-type structure** scattering features across the codebase.
- **Features importing each other's internals**, creating a tangled dependency graph.
- **A `shared/` folder that becomes a dumping ground.**

---

## 9. How an experienced engineer thinks about this

- **Test what the user experiences**: roles, labels, text, interactions, network responses.
- **Mock at the network boundary**, not inside your code.
- **Organize by feature**, with explicit public APIs and enforced boundaries.
- **Shared code earns its place** by being used by several features and having no feature
  knowledge.
- **Structure follows pain**: introduce architecture as the codebase grows, not before.

---

## 10. Check yourself

**Questions**

1. Why should frontend tests query by role and label rather than test IDs or class names?
2. What does MSW do, and why is it better than mocking `fetch` or API modules?
3. Why create a fresh `QueryClient` for each test?
4. What's wrong with large snapshot tests?
5. What are the benefits of organizing by feature instead of by technical type?
6. What dependency rules keep features independent? How can you enforce them?
7. When is a design system worth the investment?

**Exercises**

1. Write tests for the ticket detail route covering loading, 404, error and success states
   with MSW.
2. Test the new-ticket form's double-submit prevention by delaying the MSW response.
3. Restructure Beacon's `web/src` into feature folders and add an ESLint boundaries rule.
4. Write a test for `useDebouncedValue` using fake timers (`vi.useFakeTimers()`).

**Interview-style questions**

- "How do you test React components?"
- "What's the difference between unit, integration and E2E tests in a frontend?"
- "How would you structure a large React application?"

---

## 11. Going deeper

- [Testing Library documentation](https://testing-library.com/docs/react-testing-library/intro/)
  and its guide to [query priority](https://testing-library.com/docs/queries/about/#priority)
- [MSW documentation](https://mswjs.io/)
- [Vitest documentation](https://vitest.dev/)
- [Feature-Sliced Design](https://feature-sliced.design/) — one well-documented approach to
  frontend architecture.

---

## Book VI wrap-up

Beacon now has a real frontend: a mental model of rendering and reconciliation, hooks used
for what they're for, state placed deliberately (server cache, URL, local, a little global),
routing, data fetching and forms that cooperate with the API's errors, a BFF that keeps
tokens out of the browser, performance and accessibility built in, a behavior-focused test
suite, and a feature-based structure.

**Next:** [Book VII — Full-Stack Architecture](../07-fullstack/README.md) looks at how the
frontend and backend fit together as one system.
