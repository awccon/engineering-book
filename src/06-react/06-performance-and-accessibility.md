# Performance and Accessibility

Two qualities users notice immediately but developers often check last: **is it fast?**
and **can I use it?** A support agent working through a queue all day feels every 300 ms
delay. A customer who navigates by keyboard, uses a screen reader, has low vision, or is
on a slow phone either can use Beacon or can't. Both qualities are much cheaper to build
in than to retrofit, and both are increasingly measured: Core Web Vitals affect search
ranking, and accessibility is a legal requirement in many jurisdictions.

---

## 1. The problem: invisible quality

Performance and accessibility problems share a trait: developers on fast laptops with
mice and good eyesight don't experience them. The page loads instantly on localhost; the
modal works with a mouse; the gray-on-white text looks fine on a calibrated monitor. Users
on a mid-range phone over 4G, using a keyboard or a screen reader, have a different
experience. You have to deliberately measure and test.

---

## 2. Performance: the mental model

Frontend performance has two main parts:

1. **Loading**: how quickly the app becomes visible and usable (download, parse and execute
   JavaScript, fetch data, render).
2. **Runtime responsiveness**: how quickly the UI responds to interactions (typing,
   clicking, scrolling) once loaded.

### Core Web Vitals

Google's user-centric metrics, measured on real users:

| Metric | Measures | Good |
|---|---|---|
| **LCP** (Largest Contentful Paint) | When the main content appears | ≤ 2.5 s |
| **INP** (Interaction to Next Paint) | Responsiveness: delay from input to visual response, across the session | ≤ 200 ms |
| **CLS** (Cumulative Layout Shift) | Visual stability: content jumping around | ≤ 0.1 |

For an authenticated app like Beacon, INP (responsiveness) usually matters more than LCP,
because users load it once and work in it for hours.

### The main thread budget

From Book V, Chapter 2: JavaScript, layout and painting share one main thread. To feel
instant, a response to input should start painting within ~100 ms, and for smooth
animation each frame has ~16 ms. Long tasks (> 50 ms) block everything.

---

## 3. Loading performance

### Ship less JavaScript

JavaScript is the most expensive resource per byte: it must be downloaded, parsed, compiled
and executed.

- **Route-based code splitting** (Chapter 4's `lazy` routes): load each screen's code when
  it's visited.
- **Lazy-load heavy components** (rich text editor, charts) with `React.lazy` and `Suspense`.
- **Audit dependencies**: a date library, an icon pack or a utility library imported
  wholesale can add hundreds of KB. Check with `vite build` output and a bundle visualizer;
  prefer tree-shakeable imports (`import { format } from 'date-fns'`).
- **Use the platform**: `Intl.DateTimeFormat`, `Intl.RelativeTimeFormat`, `structuredClone`
  and native form validation replace many libraries.

### Cache aggressively

Hashed asset filenames (Book V, Chapter 3) can be cached forever:
`Cache-Control: public, max-age=31536000, immutable`. `index.html` must be revalidated
(`no-cache`) so new deployments are picked up. Compression (Brotli) at the server or CDN.

### Load data early

Avoid waterfalls (Chapter 4): start data requests at the route level, in parallel, or
prefetch on hover. Show skeletons with fixed dimensions to avoid layout shift.

### Images and fonts

Size images correctly, use modern formats (WebP/AVIF), lazy-load offscreen images
(`loading="lazy"`), and always set `width`/`height` to prevent layout shift. Limit custom
fonts and use `font-display: swap`.

---

## 4. Runtime performance in React

### Why React apps get slow

Most React slowness comes from **rendering too much, too often**:

- A state change high in the tree re-renders a large subtree (Chapter 1: children re-render
  with their parents).
- Each render does expensive work (sorting thousands of items, formatting dates).
- Huge lists render thousands of DOM nodes.
- Context values change too often and re-render every consumer (Chapter 3).

### Measure first

- **React DevTools Profiler**: records renders, shows which components rendered, why, and
  how long each took.
- **Chrome Performance panel**: long tasks, layout, paint; with CPU throttling (4–6×) to
  simulate a mid-range device.
- **`web-vitals` library** in production to collect real LCP/INP/CLS from users.

### Techniques

1. **Keep state local** (Chapter 3). The cheapest render is the one that doesn't happen.
2. **Move expensive work out of render or memoize it.** With the **React Compiler** enabled,
   components and values are memoized automatically: children don't re-render when their
   props are unchanged, and computed values are cached. Without the compiler, use `memo`,
   `useMemo` and `useCallback` where profiling shows a need.
3. **Virtualize long lists**: render only the visible rows (TanStack Virtual,
   react-window). 5,000 tickets become ~30 DOM nodes.
4. **Mark non-urgent updates as transitions** so input stays responsive:

```tsx
const [isPending, startTransition] = useTransition();
function onFilterChange(value: string) {
  setInputValue(value);                       // urgent: the input must update immediately
  startTransition(() => setFilter(value));    // non-urgent: re-filtering a big list can wait
}
// or: const deferredFilter = useDeferredValue(filter);
```

   React renders the urgent update first and can interrupt the transition render if the
   user keeps typing.

5. **Debounce** expensive reactions to fast input (search requests; Chapter 4).
6. **Avoid layout thrashing** in imperative code: batch DOM reads before writes.

> **🧭 When not to optimize:** If the Profiler shows a render taking 2 ms, it doesn't
> matter. Premature memoization clutters code and can even slow things down. Measure on a
> throttled CPU with realistic data volumes, fix the biggest problem, and stop.

---

## 5. Accessibility: the mental model

**Accessibility (a11y)** means people with disabilities can perceive, understand, navigate
and interact with the app. That includes people who are blind or have low vision (screen
readers, magnification), are deaf or hard of hearing, have motor impairments (keyboard-only,
switch devices, voice control), or have cognitive disabilities, plus everyone with a
temporary or situational limitation (a broken arm, bright sunlight, a noisy room).

The standard is **WCAG** (Web Content Accessibility Guidelines), organized around four
principles (**POUR**): content must be **Perceivable**, **Operable**, **Understandable** and
**Robust**. Most legal requirements (the European Accessibility Act, the ADA in the US,
Section 508) reference **WCAG 2.1 or 2.2, level AA**.

### How assistive technology sees your app

Browsers build an **accessibility tree** from the DOM: each element has a **role** (button,
link, heading, textbox), a **name** (its label), a **state** (checked, expanded, disabled)
and a **value**. Screen readers read and navigate this tree. If your "button" is a `<div>`,
it has no role, isn't focusable, and doesn't respond to Enter or Space. To assistive
technology, it isn't a button.

> **🧱 Durable:** The first rule of accessibility: **use native HTML elements for what they
> are** (`<button>`, `<a href>`, `<label>`, `<input>`, `<select>`, `<dialog>`, headings,
> lists, landmarks). They come with roles, keyboard behavior and states for free. ARIA
> attributes are for filling gaps, and "no ARIA is better than bad ARIA."

---

## 6. Accessibility in practice

### Semantics and structure

- **Landmarks**: `<header>`, `<nav>`, `<main>`, `<aside>`, `<footer>`, so screen-reader
  users can jump between regions.
- **Headings** in order (`h1` → `h2` → `h3`), describing the page structure.
- **Lists** for lists; **tables** for tabular data (with `<th scope>`).
- **Links navigate; buttons act.** Don't use a link with `onClick` for an action, or a button
  for navigation.

### Names and labels

Every interactive element needs an accessible name:

```tsx
<label htmlFor={id}>Title</label><input id={id} />            {/* visible label: best */}
<button aria-label="Close dialog"><XIcon aria-hidden /></button>   {/* icon-only button */}
<img src={avatar} alt={`${user.name}'s avatar`} />             {/* meaningful image */}
<img src={divider} alt="" />                                   {/* decorative: empty alt */}
```

Placeholder text is not a label: it disappears when typing and often has poor contrast.

### Keyboard access and focus

- Everything must work with the **keyboard**: Tab/Shift+Tab to move, Enter/Space to
  activate, Escape to close, arrow keys within composite widgets (menus, tabs, listboxes).
- **Visible focus indicators**: never `outline: none` without a replacement
  (`:focus-visible` styles).
- **Focus management in SPAs**: after client-side navigation, move focus to the new page's
  heading (or announce the change), because the browser doesn't do it for you. When a
  dialog opens, move focus into it, trap it inside, and return it to the trigger on close
  (the native `<dialog>` element with `showModal()` handles much of this).

### Dynamic updates

When content changes without a page load (a toast, a validation error, "3 new tickets"),
screen-reader users need to be told. **Live regions** announce changes:

```tsx
<div role="status" aria-live="polite">{savedMessage}</div>    {/* announced when the user is idle */}
<p role="alert">{errorMessage}</p>                            {/* announced immediately */}
```

### Forms and errors

Chapter 4's form already did this: `<label>` for every field, `aria-invalid` on invalid
fields, error text linked via `aria-describedby`, and a `role="alert"` form-level error.
Also: don't rely on color alone for errors (add text or an icon), and keep error messages
specific ("Title is required," not "Invalid input").

### Color and contrast

- Text contrast at least **4.5:1** (3:1 for large text and UI components) against its
  background.
- **Never convey information by color alone**: Beacon's status badges have text labels, and
  priority uses text plus color.
- Respect user preferences: `prefers-reduced-motion` for animations, `prefers-color-scheme`
  for dark mode, and browser zoom up to 200% without breaking layout.

### Complex widgets

For comboboxes, menus, tabs, date pickers and data grids, follow the **WAI-ARIA Authoring
Practices** patterns, or better, use an accessible headless component library (React Aria,
Radix UI, Headless UI, Ariakit) that implements the keyboard and ARIA behavior correctly.
Building an accessible combobox from scratch is weeks of work.

---

## 7. Testing accessibility

- **Automated checks** catch roughly a third of issues: `eslint-plugin-jsx-a11y` at write
  time; **axe-core** in component and end-to-end tests (`@axe-core/playwright`, Book VII);
  Lighthouse audits.
- **Keyboard testing**: put the mouse away and complete every key task.
- **Screen reader testing**: NVDA (Windows, free), VoiceOver (macOS/iOS, built in),
  TalkBack (Android). Even basic use reveals missing labels and confusing structure.
- **Zoom to 200%** and check nothing overlaps or disappears.
- **Testing Library queries** (Chapter 7) encourage accessible markup: `getByRole('button',
  { name: 'Resolve' })` only works if the button has the right role and name.

> **⚠️ What can go wrong:** Passing automated checks doesn't mean the app is accessible.
> Tools can verify that an image has `alt` text, not that the text is meaningful; that
> focus is visible, not that the focus order makes sense. Manual testing is essential.

---

## 8. In practice: making Beacon fast and accessible

### A virtualized, accessible ticket list

Agents can have thousands of tickets in a filtered view. Virtualize the list:

```bash
pnpm add @tanstack/react-virtual
```

```tsx
// web/src/tickets/VirtualTicketList.tsx
import { useVirtualizer } from '@tanstack/react-virtual';
import { useRef } from 'react';
import type { TicketSummary } from '../api/types';
import { TicketRow } from './TicketRow';

export function VirtualTicketList({ tickets, selectedId, onSelect }: {
  tickets: TicketSummary[];
  selectedId: string | null;
  onSelect: (id: TicketSummary['id']) => void;
}) {
  const parentRef = useRef<HTMLDivElement>(null);
  const virtualizer = useVirtualizer({
    count: tickets.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 56,
    overscan: 8,
  });

  return (
    <div ref={parentRef} className="ticket-scroll" tabIndex={-1}>
      <ul aria-label="Tickets" aria-rowcount={tickets.length}
          style={{ height: virtualizer.getTotalSize(), position: 'relative' }}>
        {virtualizer.getVirtualItems().map(item => {
          const ticket = tickets[item.index]!;
          return (
            <div key={ticket.id} aria-posinset={item.index + 1} aria-setsize={tickets.length}
                 style={{ position: 'absolute', top: 0, transform: `translateY(${item.start}px)`, width: '100%' }}>
              <TicketRow ticket={ticket} selected={ticket.id === selectedId} onSelect={onSelect} />
            </div>
          );
        })}
      </ul>
    </div>
  );
}
```

`aria-setsize` and `aria-posinset` tell screen readers the full list size even though only
~30 rows exist in the DOM. (Virtualization and accessibility are in some tension; for very
complex grids, consider a library built for accessible virtualized grids.)

### Responsive filtering with a transition

```tsx
const [query, setQuery] = useState('');
const deferredQuery = useDeferredValue(query);
const visible = useMemo(                    // unnecessary with the React Compiler
  () => filterLocally(tickets, deferredQuery),
  [tickets, deferredQuery],
);
const isStale = query !== deferredQuery;

<input value={query} onChange={e => setQuery(e.target.value)} aria-describedby="result-count" />
<p id="result-count" role="status">{isStale ? 'Updating…' : `${visible.length} tickets`}</p>
```

Typing stays instant; the list catches up, and the count is announced politely.

### Focus management on navigation

```tsx
// web/src/layout/RouteFocus.tsx
import { useEffect, useRef } from 'react';
import { useLocation } from 'react-router';

export function RouteHeading({ children }: { children: React.ReactNode }) {
  const ref = useRef<HTMLHeadingElement>(null);
  const { pathname } = useLocation();
  useEffect(() => { ref.current?.focus(); }, [pathname]);   // synchronize focus with the route
  return <h1 ref={ref} tabIndex={-1}>{children}</h1>;
}
```

Each page uses `<RouteHeading>` for its title, so after navigation, keyboard and
screen-reader users land at the start of the new content.

### Automated a11y checks in CI

In end-to-end tests (Book VII, Chapter 3):

```ts
import AxeBuilder from '@axe-core/playwright';

test('ticket list has no detectable accessibility violations', async ({ page }) => {
  await page.goto('/tickets');
  const results = await new AxeBuilder({ page }).withTags(['wcag2a', 'wcag2aa', 'wcag22aa']).analyze();
  expect(results.violations).toEqual([]);
});
```

### A performance budget

Add a CI check that fails if the main JavaScript chunk exceeds a size budget (for example
200 KB gzipped), and collect real-user INP with the `web-vitals` library, sent to the same
telemetry pipeline as the backend (Book IX, Chapter 6).

---

## 9. What can go wrong

- **Shipping huge bundles** because nobody looked at the build output.
- **Re-rendering the world** on every keystroke.
- **Rendering thousands of DOM nodes** instead of virtualizing.
- **Optimizing without measuring**, or measuring only on a fast laptop.
- **Divs and spans as buttons**, missing labels, icon-only buttons without names.
- **Removed focus outlines**, focus lost after navigation or dialogs.
- **Color as the only signal**, low contrast text.
- **Custom widgets without keyboard support**.
- **Trusting automated a11y checks alone.**

---

## 10. How an experienced engineer thinks about this

- **Measure on realistic devices and data**: throttled CPU, slow network, thousands of rows.
- **The fastest code is code that doesn't run**: less JavaScript, fewer renders, fewer nodes.
- **Native HTML first** for accessibility; libraries for complex widgets.
- **Keyboard and screen reader testing are part of "done"**, not a later audit.
- **Budgets and CI checks** keep both qualities from regressing.

---

## 11. Check yourself

**Questions**

1. What do LCP, INP and CLS measure? Which matters most for Beacon's agent app, and why?
2. Name four ways to reduce loading cost for a React app.
3. What are the most common causes of slow React UIs?
4. What do `useTransition` and `useDeferredValue` do?
5. What is the accessibility tree? Why is a clickable `<div>` inaccessible?
6. What's a live region, and when do you need one?
7. Why must SPAs manage focus after navigation?
8. Why aren't automated accessibility checks enough?

**Exercises**

1. Profile Beacon's ticket page with 5,000 tickets and CPU throttling; fix the slowest
   interaction.
2. Navigate the whole app with only the keyboard and fix every blocker you find.
3. Use NVDA or VoiceOver to create a ticket. Note what's confusing and fix it.
4. Add a bundle-size budget check to CI.

**Interview-style questions**

- "How would you improve the performance of a slow React application?"
- "What are Core Web Vitals?"
- "How do you make a web application accessible?"
- "How do you make a custom dropdown accessible?"

---

## 12. Going deeper

- [web.dev: Core Web Vitals](https://web.dev/articles/vitals) and [Learn Performance](https://web.dev/learn/performance)
- [react.dev: React Compiler](https://react.dev/learn/react-compiler) and the
  [Profiler](https://react.dev/reference/react/Profiler)
- [WCAG 2.2 quick reference](https://www.w3.org/WAI/WCAG22/quickref/) and the
  [ARIA Authoring Practices Guide](https://www.w3.org/WAI/ARIA/apg/)
- [The A11Y Project checklist](https://www.a11yproject.com/checklist/)

**Next:** [Chapter 7 — Testing and Frontend Architecture](07-testing-and-frontend-architecture.md)
tests Beacon's frontend and organizes it to grow.
