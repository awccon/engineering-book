# End-to-End Testing and Environments

Every layer of Beacon has its own tests: domain unit tests, API integration tests against
real PostgreSQL, component tests with mocked network. Each one replaces something real
with something controlled. None of them proves that a real user, in a real browser, can
sign in through the real identity flow, create a ticket through the BFF and the API, and
see it appear for another agent in real time.

That's the job of **end-to-end (E2E) tests**, and of the **environments** they run in. This
chapter covers both: how to write E2E tests that are valuable rather than flaky, and how to
structure local, preview, staging and production environments.

---

## 1. The problem: the pieces work; does the system?

Integration failures that only E2E tests (or users) catch:

- The BFF proxies `/api` but not `/hubs`, so live updates never connect.
- A cookie attribute that works on `localhost` fails on the real domain.
- The SPA fallback route returns `index.html` for a missing JavaScript chunk.
- A CSP header blocks a script in production but not in development.
- A migration ran, but the API deployed before it (or after it) in the wrong order.

E2E tests exercise the deployed system the way users do. They're slower and more fragile
than other tests, so the art is choosing *few, valuable* ones and making them reliable.

---

## 2. The mental model: the test portfolio, revisited

```text
               ▲ E2E (Playwright)                 few: critical user journeys, smoke tests after deploy
              ▲▲▲ Integration (API + DB; components + MSW)    many: behavior at the seams
            ▲▲▲▲▲▲▲ Unit (domain, reducers, pure functions)   many: rules and edge cases
```

| Question | Best answered by |
|---|---|
| Does `Ticket.Resolve` reject closed tickets? | Unit test |
| Does `POST /api/tickets/{id}/comments` return 409 for closed tickets? | API integration test |
| Does the form show the 409 message correctly? | Component test with MSW |
| Can an agent sign in, open their queue, comment, and resolve a ticket, with a customer seeing the update? | **E2E test** |

E2E tests should cover **user journeys that matter most to the business** and **the wiring
between deployed components**, not every edge case (those are cheaper and more precise
lower down).

---

## 3. Playwright

**Playwright** drives real browsers (Chromium, Firefox, WebKit) and has become the standard
E2E tool. Features that make it reliable:

- **Auto-waiting**: actions wait for elements to be visible, enabled and stable; assertions
  retry until they pass or time out. No `sleep(2000)`.
- **Web-first assertions**: `await expect(locator).toHaveText(...)`.
- **Role-based locators**, like Testing Library: `page.getByRole('button', { name: 'Resolve' })`.
- **Isolation**: each test gets a fresh **browser context** (like a new incognito profile).
- **Tracing**: on failure, a trace file records every action, DOM snapshot, network request
  and console message, viewable in the trace viewer. This makes failures in CI diagnosable.
- **Multiple users**: several browser contexts in one test (agent and customer side by side).

```bash
cd e2e
pnpm create playwright
```

```ts
// e2e/playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  retries: process.env.CI ? 1 : 0,
  reporter: [['html'], ['list']],
  use: {
    baseURL: process.env.BASE_URL ?? 'https://localhost:5001',
    trace: 'retain-on-failure',
    screenshot: 'only-on-failure',
  },
  projects: [
    { name: 'setup', testMatch: /auth\.setup\.ts/ },
    { name: 'chromium', use: { ...devices['Desktop Chrome'] }, dependencies: ['setup'] },
    { name: 'webkit', use: { ...devices['Desktop Safari'] }, dependencies: ['setup'] },
  ],
});
```

---

## 4. Making E2E tests reliable

Flaky E2E tests are the main reason teams give up on them. The causes and cures:

| Cause of flakiness | Cure |
|---|---|
| Fixed sleeps and timing assumptions | Auto-waiting locators and web-first assertions |
| Tests depending on each other or on shared data | Each test creates its own data (unique titles, fresh tickets) |
| Logging in through the UI in every test | Log in once per role, save the session state, reuse it |
| Real third-party services (email, payment) | Fakes or sandboxes in test environments; route interception for edge cases |
| Animations, transitions | Wait for final state; `prefers-reduced-motion` in tests |
| Environment drift (different config than production) | Same container images and configuration shape everywhere (Books VIII–IX) |
| Brittle selectors (CSS classes, DOM structure) | Role, label and text locators |

> **⚠️ What can go wrong:** Retrying flaky tests until they pass hides real
> race conditions, and some of those are bugs users will hit. Track flaky tests explicitly,
> fix or quarantine them quickly, and treat a test that "sometimes fails" as a bug report.

### Authentication without the UI every time

Sign in through the real flow **once per role** in a setup project, save the browser storage
(cookies), and reuse it:

```ts
// e2e/tests/auth.setup.ts
import { test as setup, expect } from '@playwright/test';

for (const role of ['agent', 'customer'] as const) {
  setup(`authenticate as ${role}`, async ({ page }) => {
    await page.goto('/bff/login');
    await page.getByLabel('Username').fill(process.env[`E2E_${role.toUpperCase()}_USER`]!);
    await page.getByLabel('Password').fill(process.env[`E2E_${role.toUpperCase()}_PASSWORD`]!);
    await page.getByRole('button', { name: 'Sign in' }).click();
    await expect(page.getByRole('heading', { level: 1 })).toBeVisible();
    await page.context().storageState({ path: `.auth/${role}.json` });
  });
}
```

Test users live in a test identity provider (a Keycloak container locally, a test tenant in
staging) with credentials from secrets, never from the repository.

### Test data

Create data through the **API** (fast, reliable), not the UI, unless creating it *is* the
journey being tested. Use unique values so tests can run in parallel and against shared
environments:

```ts
const title = `E2E VPN outage ${crypto.randomUUID().slice(0, 8)}`;
```

---

## 5. In practice: Beacon's critical journeys

Beacon's E2E suite covers a handful of journeys:

1. A customer creates a ticket and sees it in "My tickets."
2. An agent sees it in the queue, comments, and resolves it; the **customer sees the update
   live**.
3. A customer can't see another customer's ticket (404), even by URL.
4. Session expiry redirects to login and returns to the same page.
5. Accessibility scan of the main pages (Book VI, Chapter 6).

The second journey, with two users at once:

```ts
// e2e/tests/ticket-lifecycle.spec.ts
import { test, expect } from '@playwright/test';

test('agent resolves a customer ticket and the customer sees it live', async ({ browser }) => {
  const customer = await browser.newContext({ storageState: '.auth/customer.json' });
  const agent = await browser.newContext({ storageState: '.auth/agent.json' });
  const customerPage = await customer.newPage();
  const agentPage = await agent.newPage();

  const title = `E2E printer offline ${crypto.randomUUID().slice(0, 8)}`;

  // Customer creates a ticket through the UI (this IS the journey)
  await customerPage.goto('/tickets/new');
  await customerPage.getByLabel('Title').fill(title);
  await customerPage.getByLabel('Priority').selectOption('High');
  await customerPage.getByRole('button', { name: 'Create ticket' }).click();
  await expect(customerPage.getByRole('heading', { name: title })).toBeVisible();
  await expect(customerPage.getByText('Open')).toBeVisible();

  // Agent finds it in the queue and resolves it
  await agentPage.goto(`/tickets?q=${encodeURIComponent(title)}`);
  await agentPage.getByRole('button', { name: new RegExp(title) }).click();
  await agentPage.getByLabel('Add a comment').fill('Replaced the toner; printing again.');
  await agentPage.getByRole('button', { name: 'Post comment' }).click();
  await agentPage.getByRole('button', { name: 'Resolve' }).click();
  await expect(agentPage.getByText('Resolved')).toBeVisible();

  // Customer's open page updates without a reload (SignalR through the BFF)
  await expect(customerPage.getByText('Resolved')).toBeVisible({ timeout: 10_000 });
  await expect(customerPage.getByText('Replaced the toner; printing again.')).toBeVisible();

  await customer.close();
  await agent.close();
});
```

This single test verifies the BFF login flow (via the saved sessions), routing, the form,
the API, the database, authorization for two roles, the outbox, SignalR proxied through the
BFF, and the React cache update, in about ten seconds. That's the value of a well-chosen E2E
test.

The authorization journey is short and important:

```ts
test('customers cannot open other customers\' tickets', async ({ browser }) => {
  const other = await browser.newContext({ storageState: '.auth/customer2.json' });
  const page = await other.newPage();
  const ticketId = await createTicketViaApi('customer');     // helper using the API with the first customer's token
  await page.goto(`/tickets/${ticketId}`);
  await expect(page.getByRole('heading', { name: 'Ticket not found' })).toBeVisible();
});
```

---

## 6. Environments

### The mental model: a path to production

```text
 Local  ──►  PR preview  ──►  Staging  ──►  Production
 (dev)       (per pull        (shared,       (real users)
              request)         prod-like)
```

| Environment | Purpose | Data | Who uses it |
|---|---|---|---|
| **Local** | Develop and debug | Generated seed data | One developer |
| **CI** | Build, unit/integration tests | Containers created per run (Testcontainers) | Pipeline |
| **Preview** (ephemeral) | Review a PR running for real; run E2E | Seed data, created and destroyed with the PR | Reviewers, PMs, E2E |
| **Staging** | Final verification in a production-like setup | Production-shaped, anonymized or generated | Team, E2E, load tests |
| **Production** | Real users | Real data | Everyone |

Principles:

- **Production parity**: same container images, same configuration *shape* (different
  values), same infrastructure definitions (Book IX), same database engine and version.
  "Works in staging" should mean something.
- **Build once, deploy many**: the artifact tested in CI and staging is exactly the one
  promoted to production. Configuration differs; code doesn't (Book III, Chapter 3).
- **No production data in non-production environments** (Book IV, Chapter 8).
- **Ephemeral environments** for previews are now practical with containers and
  infrastructure as code, and are excellent for reviews and E2E runs isolated from
  other work.

### Local development for a multi-service system

Beacon locally means PostgreSQL, an identity provider, Beacon.Api, Beacon.Bff and the Vite
dev server. Running them by hand is tedious. Options:

- **Docker Compose** for infrastructure (PostgreSQL, Keycloak, a fake SMTP server like
  Mailpit), with the .NET and Node processes run from the IDE (Book VIII, Chapter 5).
- **.NET Aspire**: an orchestration layer that starts the projects and containers together,
  wires connection strings, and shows logs and traces in one dashboard.

> **🔄 Current (as of October 2026):** .NET Aspire (now often just "Aspire") is Microsoft's
> recommended way to orchestrate multi-project .NET apps locally, including containers and
> Node/Vite apps, with built-in OpenTelemetry dashboards. It can also generate deployment
> artifacts. Docker Compose remains the portable, tool-agnostic option.

### Smoke tests after deployment

After every deployment to staging and production, run a **small** subset of E2E tests (or
synthetic checks) against the live environment: sign in, load the queue, open a ticket.
They catch configuration and infrastructure problems that no pre-deployment test can,
and can trigger automatic rollback (Book IX, Chapter 9).

---

## 7. Running E2E in CI

```yaml
# .github/workflows/e2e.yml (sketch)
name: E2E
on: [pull_request]
jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker compose -f docker-compose.e2e.yml up -d --build --wait   # db, keycloak, api, bff (with built SPA)
      - uses: actions/setup-node@v4
        with: { node-version: 24 }
      - run: pnpm --dir e2e install --frozen-lockfile
      - run: pnpm --dir e2e exec playwright install --with-deps chromium webkit
      - run: pnpm --dir e2e exec playwright test
        env:
          BASE_URL: https://localhost:5001
          E2E_AGENT_USER: agent1
          E2E_AGENT_PASSWORD: ${{ secrets.E2E_AGENT_PASSWORD }}
      - uses: actions/upload-artifact@v4
        if: failure()
        with: { name: playwright-report, path: e2e/playwright-report }
```

The uploaded report includes traces for failures, so a red build is diagnosable without
reproducing it locally.

> **🧭 When not to add an E2E test:** If a behavior can be verified reliably by a unit,
> API or component test, test it there. Each E2E test costs seconds of run time and a
> chance of flakiness, multiplied by every run. Reserve them for journeys and wiring.

---

## 8. What can go wrong

- **Too many E2E tests**, slow and flaky, until the team ignores failures.
- **Tests sharing data**, failing when run in parallel or in a different order.
- **UI login in every test.**
- **Sleeps instead of waiting for conditions.**
- **Environments that don't match production**, so passing tests prove little.
- **Production data copied to staging.**
- **No post-deployment smoke tests**, so configuration errors reach users first.
- **Unreadable failures** without traces, screenshots or logs.

---

## 9. How an experienced engineer thinks about this

- **E2E for journeys and wiring; lower levels for logic and edge cases.**
- **Reliability is a feature of the test suite**; flaky tests are bugs.
- **Production parity is the point of non-production environments.**
- **Build once, configure per environment, promote the same artifact.**
- **Verify after deploying**, not only before.

---

## 10. Check yourself

**Questions**

1. What kinds of failures can only E2E tests catch?
2. What makes Playwright tests more reliable than older E2E tools?
3. Name four causes of flaky E2E tests and their cures.
4. Why save authenticated session state instead of logging in through the UI in every test?
5. What's the purpose of each environment from local to production?
6. What does "build once, deploy many" mean?
7. What are smoke tests, and when do they run?

**Exercises**

1. Set up Playwright for Beacon with the auth setup project and write the "customer creates a
   ticket" journey.
2. Write the two-user live-update test and run it with tracing; open the trace viewer.
3. Create `docker-compose.e2e.yml` that starts everything needed for the E2E suite.
4. Add an axe accessibility scan to two E2E tests.

**Interview-style questions**

- "How do you decide what to cover with end-to-end tests?"
- "How do you deal with flaky tests?"
- "Describe the environments you'd set up for a web application, and why."

---

## 11. Going deeper

- [Playwright documentation](https://playwright.dev/) — especially "Best Practices" and
  "Authentication."
- [.NET Aspire documentation](https://learn.microsoft.com/dotnet/aspire/)
- Martin Fowler, [The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html)

---

## Book VII wrap-up

Beacon is now one system rather than a collection of layers: requests traced end to end,
logic placed where it has authority, screen-oriented endpoints and server-computed
permissions, a generated contract shared by backend and frontend with CI gates, and E2E
tests that verify the critical journeys across every component.

Everything so far has run on a developer machine. The next books put it on servers.

**Next:** [Book VIII — Docker & Linux](../08-docker-linux/README.md).
