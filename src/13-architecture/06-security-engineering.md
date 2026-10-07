# Security Engineering

Security has appeared in almost every book: parameterized queries and database roles (Book IV), XSS
and CSRF in the browser (Books V–VI), JWTs, cookies and the BFF (Books III and VI), hardened containers
(Book VIII), managed identities and private networking (Book IX), prompt injection (Book XI), and SSRF
in `beacon-relay` (Book XII). Those chapters taught **controls**. This chapter teaches the **engineering
discipline** that decides which controls a system needs: thinking like an attacker, threat modeling,
defense in depth, protecting data and tenants, and securing the software supply chain that builds and
ships your code.

---

## 1. The problem: security is a property of the whole system

A system is as secure as its weakest reachable path. A perfectly implemented authentication flow doesn't
help if an export endpoint forgets the tenant filter, a debug endpoint is left enabled, a CI token can
push to `main`, or an npm package used by the build is compromised. Attackers don't attack your design
document; they look for the one path you didn't think about.

Three things make security different from other quality attributes:

1. **There's an adversary.** Bugs from bad luck are random; attackers search deliberately for the worst
   case, and they share techniques.
2. **Failures are often invisible.** A data breach doesn't raise the error rate. You may learn about it
   months later, from someone else.
3. **Consequences are asymmetric.** One flaw can expose every customer's data, and you can't roll back a
   leak.

So security can't be a feature added at the end or a scan run before release. It has to be part of how
a system is designed, built, shipped and operated.

---

## 2. The mental model: assets, trust boundaries and attackers

Security engineering starts with three questions:

- **What are we protecting?** (assets): customer data, credentials, the ability to act as a user, the
  service's availability, the integrity of the code we ship, the company's money.
- **Where does trust change?** (trust boundaries): every point where data or control crosses from a less
  trusted to a more trusted zone. The browser → BFF. The internet → `beacon-relay`'s outbound calls. A
  customer's ticket text → the AI model's prompt. A pull request from a fork → the CI runner. A
  third-party package → your build.
- **Who might attack, and how?** (threat actors): opportunistic scanners, malicious customers (other
  tenants), compromised user accounts, malicious insiders, supply-chain attackers, and occasionally
  targeted, well-funded groups.

> **🧱 Durable:** **Every input that crosses a trust boundary is hostile until validated, and every
> component that can be reached should be assumed reachable by an attacker.** Most vulnerabilities are a
> trust boundary someone didn't notice: data treated as trusted because it came "from our own database",
> "from our own frontend" or "from the model".

### Core principles

These principles are decades old and still the best guide:

- **Least privilege**: every user, service and process gets only the permissions it needs, for only as
  long as it needs them. Beacon's managed identities have narrow roles (Book IX); `beacon-relay`'s
  database role can read only the outbox (Chapter 2).
- **Defense in depth**: multiple independent layers, so one failure isn't a breach. Authorization in the
  API **and** row-level security in the database; a WAF **and** input validation **and** output encoding.
- **Secure defaults / fail closed**: when in doubt, deny. New endpoints require authorization unless
  explicitly marked anonymous (`FallbackPolicy`, Book III).
- **Minimize attack surface**: every endpoint, port, dependency, permission and feature is something to
  defend. Remove what you don't use.
- **Separation of duties**: no single person or credential can both make and approve a high-risk change.
- **Zero trust**: don't trust network location. Authenticate and authorize every request, even inside the
  "private" network.
- **Don't roll your own crypto, auth or sanitizers**: use well-reviewed libraries and platform features.

---

## 3. Threat modeling

**Threat modeling** is structured thinking about what can go wrong, done while the design can still
change cheaply. Adam Shostack's four questions are the whole method:

1. **What are we working on?** Draw a data flow diagram: processes, data stores, external entities, data
   flows, and **trust boundaries** as dashed lines.
2. **What can go wrong?** Walk each element and flow with a checklist such as **STRIDE**.
3. **What are we going to do about it?** Mitigate, accept, transfer or eliminate each threat.
4. **Did we do a good job?** Review: were the mitigations built, tested and effective?

### STRIDE

| Threat | Violates | Question to ask | Typical mitigations |
|---|---|---|---|
| **S**poofing | Authentication | Can someone pretend to be another user or service? | Strong auth, MFA, signed tokens, mTLS/managed identity |
| **T**ampering | Integrity | Can someone modify data or code in transit or at rest? | TLS, signatures (webhooks, artifacts), authorization on writes, immutable logs |
| **R**epudiation | Non-repudiation | Can someone deny an action because there's no evidence? | Audit logs with actor, time, before/after |
| **I**nformation disclosure | Confidentiality | Can someone read data they shouldn't? | Authorization, tenant isolation, encryption, minimal responses, safe errors |
| **D**enial of service | Availability | Can someone exhaust resources? | Rate limits, quotas, size limits, timeouts, autoscaling with caps |
| **E**levation of privilege | Authorization | Can someone gain rights they shouldn't have? | Server-side authorization on every request, least privilege, sandboxing |

Threat modeling doesn't need special tools or days of meetings. For a feature, an hour with a
whiteboard, the developers, and someone who thinks adversarially finds most of the important issues.
Microsoft's Threat Modeling Tool and OWASP Threat Dragon help for larger systems, and keeping the diagram
in the repo (Mermaid) keeps it current.

### In practice: threat modeling Beacon's customer webhooks

Customers can register HTTPS endpoints to receive ticket events (delivered by `beacon-relay`, Book XII).

```text
            trust boundary                     trust boundary
 Customer admin ┊──► Beacon API ──► webhooks table ──► outbox ──► beacon-relay ┊──► Customer endpoint
  (browser)     ┊    (register)     (url, secret)                 (signs, POSTs)┊    (internet)
```

Walking STRIDE produced this list (abbreviated):

| # | Threat | Mitigation | Status |
|---|---|---|---|
| 1 | **E/I**: Endpoint URL points to internal addresses (`http://169.254.169.254`, `10.x`, `localhost`): SSRF to cloud metadata or internal services | Validate at registration **and** at every delivery after DNS resolution; block private, loopback, link-local ranges; HTTPS only; relay runs in a subnet with no route to internal services | Done (Book XII) + network rule added |
| 2 | **S**: Attacker sends fake webhooks to a customer, pretending to be Beacon | HMAC signature with per-endpoint secret; timestamp in signed payload to prevent replay | Done; timestamp added |
| 3 | **I**: Events for tenant A delivered to tenant B's endpoint | Endpoint lookup by tenant ID from the event, test coverage, RLS on webhook table | Added test |
| 4 | **I**: Payload includes data the customer's endpoint shouldn't get (internal notes, agent emails) | Explicit webhook payload contract; internal notes excluded; contract test | Done |
| 5 | **D**: Customer registers a slow endpoint to tie up relay capacity | Per-endpoint concurrency limits, timeouts, automatic disabling after repeated failures | Done (Book XII) |
| 6 | **R**: Customer claims they never registered an endpoint that leaked data | Audit log for webhook create/update/delete with actor | Added |
| 7 | **I**: Webhook secrets readable by support staff in the admin UI or logs | Secrets shown once at creation; stored encrypted; redacted in logs (`[Sensitive]`, Book I) | Done |
| 8 | **E**: Redirects to internal addresses | Relay doesn't follow redirects | Done |

Notice that several mitigations already existed because earlier chapters thought about them, but items 2
(replay), 3 and 6 were gaps found only by the systematic walk.

---

## 4. Defense in depth in Beacon

Here is Beacon's security architecture as layers. Each one assumes the others might fail.

```text
Edge          Front Door + WAF: TLS, managed rules, rate limits, bot protection, geo rules if needed
Identity      Entra ID (agents, MFA, conditional access) / Entra External ID (customers); BFF cookie session
Application   Authorization policies on every endpoint (fallback: require auth), resource-based checks,
              input validation, output encoding, Problem Details without internals, rate limits per tenant
Data          Tenant filters in queries + PostgreSQL row-level security; least-privilege DB roles;
              encryption at rest; column-level protection for secrets
Network       Private endpoints, no public DB/Redis/Service Bus; relay egress isolated from VNet
Workload      Non-root, read-only containers, minimal images, managed identity, no secrets in env vars
Supply chain  Locked dependencies, scanning, pinned actions, signed images, SBOMs, protected branches
Detection     Audit logs, security alerts, anomaly detection on auth and exports, Defender for Cloud
Response      Runbooks, key rotation procedures, the ability to revoke sessions and disable features fast
```

### Multi-tenant isolation

For a SaaS application, the most damaging common vulnerability is **cross-tenant data access**: one
customer seeing another's tickets because a query forgot a filter. It's a broken-access-control bug, the
top category in the OWASP Top 10 for years. Defense in depth applies:

**Layer 1: the application filters by tenant.** EF Core global query filters apply the tenant to every
query on tenant-owned entities, so forgetting the filter requires actively disabling it:

```csharp
internal sealed class TicketsDbContext(DbContextOptions<TicketsDbContext> options, ITenantContext tenant)
    : DbContext(options)
{
    protected override void OnModelCreating(ModelBuilder model)
    {
        model.Entity<Ticket>().HasQueryFilter(t => t.OrganizationId == tenant.OrganizationId);
        // ... same for comments, attachments, webhooks
    }
}
```

**Layer 2: the database enforces it too**, with row-level security (Book IV, Chapter 3), so a raw SQL
query, a reporting job or a future bug can't cross tenants:

```sql
ALTER TABLE tickets.tickets ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON tickets.tickets
    USING (organization_id = current_setting('beacon.tenant_id', true)::uuid)
    WITH CHECK (organization_id = current_setting('beacon.tenant_id', true)::uuid);
-- beacon_app is not the table owner, so the policy applies to it.
-- If the setting is missing, current_setting(..., true) returns NULL and no rows match: fail closed.
```

```csharp
// Sets the tenant on every connection the app opens; pooled connections get it reset on each open
internal sealed class TenantConnectionInterceptor(ITenantContext tenant) : DbConnectionInterceptor
{
    public override async Task ConnectionOpenedAsync(DbConnection connection, ConnectionEndEventData data,
        CancellationToken ct = default)
    {
        await using var cmd = connection.CreateCommand();
        cmd.CommandText = "SELECT set_config('beacon.tenant_id', @t, false)";
        var p = cmd.CreateParameter();
        p.ParameterName = "t";
        p.Value = tenant.OrganizationId.ToString();
        cmd.Parameters.Add(p);
        await cmd.ExecuteNonQueryAsync(ct);
    }
}
```

**Layer 3: tests that try to break it.** An integration test suite creates two tenants and, for every
endpoint, asserts that tenant A's credentials can't read, list, search, export or modify tenant B's data.
Generate the test cases from the endpoint list so new endpoints are covered automatically.

**Layer 4: caches, search and AI respect it.** Cache keys include the tenant (Book III, Chapter 7);
vector search filters by tenant **inside** the query, not after (Book XI, Chapter 6); background jobs set
the tenant context explicitly per message.

> **⚠️ What can go wrong:** Tenant isolation fails most often outside the main request path: a CSV export
> job, a search index shared across tenants, a cache key without the tenant, an AI assistant whose
> retrieval step searches all documents, an admin "impersonate" feature without audit, or a background
> job that processes messages for many tenants with one connection whose tenant setting was left from
> the previous message.

### Protecting data

- **Classify data**: public, internal, confidential (customer ticket content), restricted (credentials,
  secrets, payment data, special categories of personal data). Controls follow classification.
- **Minimize**: don't collect, log, send to third parties (including AI providers) or retain what you
  don't need. Data you don't have can't leak.
- **Encrypt** in transit (TLS everywhere, including inside the VNet) and at rest (platform default). For
  restricted fields such as webhook secrets, add application-level encryption with keys in Key Vault, so a
  database dump alone doesn't reveal them.
- **Retention and deletion**: privacy laws (GDPR and others) give people rights to access and delete their
  data. Design deletion from the start: know every place personal data goes (database, backups, search
  index, embeddings, logs, analytics, AI provider), and how it's removed or expires. Beacon's
  `beacon-retention` job (Book X) handles the main stores; backups expire on schedule.
- **Logs are data too**: redact secrets and personal data (Book I's `[Sensitive]` attribute, Book III's
  logging setup), restrict access, and set retention.

### Secrets

The pattern, repeated from Book IX because it matters: **prefer no secrets** (managed identity, workload
identity federation for CI), and keep unavoidable ones in Key Vault with rotation, never in source code,
images, environment files committed to Git or logs. Enable **secret scanning** with push protection on the
repository; treat any secret that reaches Git history as compromised and rotate it.

---

## 5. The software supply chain

Your application contains far more code you didn't write than code you did: NuGet and npm packages, base
images, GitHub Actions, build tools, and the AI coding assistants that suggest code (Book XI, Chapter 11).
Attackers increasingly target that chain, because compromising one popular package or build step reaches
thousands of victims.

Recent incidents show the range:

- **SolarWinds (2020)**: attackers compromised the build system and inserted a backdoor into signed
  releases.
- **Log4Shell (2021)**: a critical vulnerability in a ubiquitous logging library; the hard part for most
  organizations was finding out **where** they used it.
- **xz-utils (2024)**: a multi-year social-engineering effort gave an attacker maintainer access to a
  compression library and a backdoor that nearly shipped in major Linux distributions.

> **🔄 Current (as of October 2026):** The npm ecosystem saw repeated large compromises in 2025, including
> the hijacking of widely used packages through phished maintainer accounts and self-propagating malware
> that stole tokens from developer machines and CI and used them to publish further infected packages.
> GitHub Actions saw the compromise of popular third-party actions (for example `tj-actions/changed-files`
> in March 2025) that exposed CI secrets in logs. Registries responded with stricter publishing
> requirements (trusted publishing via OIDC, mandatory 2FA, shorter-lived tokens). Expect this area to
> keep changing; follow your package registries' security announcements.

### Controls that matter

**Dependencies**

- **Lock files committed and enforced**: `packages.lock.json` with `RestoreLockedMode` in CI for .NET,
  `pnpm-lock.yaml` with `--frozen-lockfile`, `uv.lock`, `Cargo.lock` for applications.
- **Fewer dependencies**: each package is a trust decision. Prefer the platform's standard library and
  well-maintained, widely used packages. A 10-line helper doesn't need a dependency.
- **Vulnerability scanning**: `dotnet list package --vulnerable`, `pnpm audit`, `cargo audit`, `pip-audit`,
  Dependabot or Renovate alerts, with a process to triage and update.
- **Update regularly and in small steps** with automated PRs and good tests, so a critical patch is a
  routine merge rather than a migration.
- **Delay brand-new versions**: a short cooldown before adopting a just-published version (Renovate's
  `minimumReleaseAge`, pnpm's `minimumReleaseAge` setting) avoids most malicious releases, which are
  usually detected and removed within days.
- **Disable install scripts** where possible (`pnpm` doesn't run dependency lifecycle scripts unless
  allowed; review `onlyBuiltDependencies`), because install scripts are a common malware vector.
- **Use a private feed or proxy** with package source mapping (`nuget.config` `packageSourceMapping`) to
  prevent dependency confusion attacks, where a public package with your internal package's name gets
  installed.

**Build pipeline**

- **Pin third-party actions by commit SHA**, not tag: tags can be moved to malicious commits.
- **Minimal `permissions:`** on every workflow (`contents: read` by default) and per-job elevation.
- **OIDC federation** to Azure instead of stored cloud credentials (Book IX, Chapter 8).
- **Never run untrusted code with secrets**: be extremely careful with `pull_request_target` and
  `workflow_run` triggers, which run with repository secrets in the context of fork PRs.
- **Protected branches and environments**: required reviews, required status checks, no direct pushes,
  deployment approvals for production.
- **Ephemeral runners** for sensitive builds so nothing persists between jobs.

**Artifacts**

- **SBOMs** (software bills of materials, in SPDX or CycloneDX format) generated at build time, so that
  the next Log4Shell is a query ("which images contain this package?"), not a week of searching.
- **Provenance and signing**: build attestations (GitHub artifact attestations, Sigstore/cosign) prove
  that an image was built by your pipeline from a specific commit. The **SLSA** framework defines levels
  of build integrity to aim for.
- **Deploy by digest** and verify signatures before deployment (Book IX already deploys by digest).

---

## 6. A secure development lifecycle that people actually follow

Security activities that work are **built into the normal workflow** rather than added as gates at the
end:

| Stage | Activity | Beacon example |
|---|---|---|
| Design | Threat model for features that touch trust boundaries, auth, data or money | Webhooks (above), AI agent tools (Book XI) |
| Code | Secure defaults in templates; security-focused review checklist for risky areas | Fallback auth policy; review checklist in PR template |
| Commit | Secret scanning with push protection | GitHub secret scanning |
| Build | SAST (CodeQL), dependency and container scanning, IaC scanning, lock files | CodeQL, Dependabot, Trivy, Bicep linter rules |
| Test | Authorization and tenant isolation tests; DAST against staging | Cross-tenant suite; OWASP ZAP baseline scan nightly |
| Release | Signed artifacts, provenance, approvals | Attestations, protected `production` environment |
| Operate | Security monitoring, patching, access reviews, key rotation | Defender for Cloud, quarterly access review |
| Respond | Incident runbooks, contact points, disclosure policy | `security.txt`, runbook for credential leak |
| Learn | Penetration tests, bug bounty, postmortems | Annual pentest; findings tracked like bugs |

Scanners produce noise. A process that works triages findings by **reachability and exploitability**,
fixes the important ones quickly, documents accepted risks, and tunes rules so developers trust the
remaining alerts.

### Security incident response

Prepare for the day something goes wrong:

- **Know how to revoke**: user sessions (BFF session store), tokens, API keys, webhook secrets, deploy
  credentials, and how quickly each can be rotated.
- **Keep audit logs** that answer "who accessed what, when" (and protect them from the people they
  audit).
- **Have a runbook** for the likely scenarios: leaked secret, compromised account, vulnerable dependency
  under active exploitation, data exposed to the wrong tenant.
- **Know your obligations**: breach notification laws can require notifying regulators and customers
  within days (for example, 72 hours to the regulator under GDPR). Involve legal early.

---

## 7. The OWASP Top 10, mapped to this book

The OWASP Top 10 for web applications is the most widely used summary of common risks. Its categories
shift slightly between editions; the underlying issues are stable. Where this book covers each:

| Risk area | Where covered |
|---|---|
| Broken access control (incl. IDOR, cross-tenant access, SSRF) | Book III, Ch 6; this chapter, sections 3–4; Book XII, Ch 4 |
| Security misconfiguration | Book III, Ch 9; Book VIII, Ch 6; Book IX, Ch 2 |
| Software supply chain failures, vulnerable components | This chapter, section 5 |
| Cryptographic failures | Book III, Ch 9 (TLS, hashing); this chapter (data protection) |
| Injection (SQL, command, XSS) | Book IV, Ch 2 and 8; Book VI, Ch 5 |
| Insecure design | This chapter, section 3 (threat modeling) |
| Authentication failures | Book III, Ch 6; Book VI, Ch 5 |
| Data integrity failures (unsigned updates, unsafe deserialization) | Book III, Ch 5; webhook signing |
| Logging and alerting failures | Book IX, Ch 6; this chapter, section 6 |
| Mishandling of exceptional conditions (failing open, leaking errors) | Book I, Ch 8; Book III, Ch 10; Chapter 3 |

> **🔄 Current (as of October 2026):** OWASP published a 2025 edition of the web Top 10, which elevated
> software supply chain failures and added the mishandling of exceptional conditions as categories.
> OWASP also maintains a Top 10 for LLM applications (prompt injection, sensitive information disclosure,
> excessive agency and others; Book XI, Chapter 9) and an API Security Top 10. Read the current lists
> rather than memorizing the rankings.

---

## 8. What can go wrong

- **Security as a final gate**: a pentest a week before launch finds design flaws that are expensive to
  fix, and they get "accepted".
- **Trusting the frontend**: hiding a button instead of enforcing authorization on the server.
- **Authorization by obscurity**: unguessable IDs used instead of access checks.
- **Inconsistent enforcement**: checks in most endpoints, missing in the export, the bulk operation, the
  GraphQL resolver, the SignalR hub method, or the AI agent's tools.
- **Over-privileged identities**: a service principal with Owner on the subscription "to make it work".
- **Secrets in the wrong places**: Git history, container images, CI logs, crash dumps, AI prompts.
- **Unpinned, unreviewed dependencies and actions**, with install scripts running on developer machines
  and CI runners that hold credentials.
- **Alert fatigue** from unprioritized scanner output, so real findings are ignored.
- **No inventory**: when a critical vulnerability is announced, nobody knows which services are affected.
- **Logging too much**: tokens, passwords, personal data in logs readable by many people.

---

## 9. When not to use it

> **🧭 When not to use it:** Security controls have costs: complexity, friction, money and time. Match
> them to risk. An internal prototype with no real data doesn't need a full threat model, a pentest and
> signed builds. A public, multi-tenant SaaS handling customer data needs all of them. What you should
> **never** skip, at any size: server-side authorization, parameterized queries, output encoding, TLS,
> secrets out of source control, and keeping dependencies updated. Also avoid security theater: controls
> that look strict but don't address real threats (forced password rotation every 30 days, blanket
> blocking of copy/paste, scanner gates nobody triages) cost goodwill and protect little.

---

## 10. How an experienced engineer thinks about this

- **Asks "how would I abuse this?"** for every feature, especially at trust boundaries.
- **Threat models early and lightly**, while the design can still change, and keeps the model in the
  repo.
- **Layers defenses** so one bug isn't a breach: application checks plus database enforcement plus tests.
- **Makes the secure path the easy path**: templates, defaults, fallback policies, shared libraries.
- **Treats dependencies as trust decisions** and the build pipeline as production infrastructure.
- **Minimizes data**: what isn't collected, logged or sent can't leak.
- **Plans for compromise**: revocation, rotation, audit logs and runbooks before they're needed.
- **Prioritizes by risk**, fixing exploitable issues fast and declining to chase noise.

---

## 11. Check yourself

**Questions**

1. What are trust boundaries? Name five in Beacon.
2. List and explain the six STRIDE categories with a mitigation for each.
3. What are Shostack's four threat-modeling questions?
4. What's defense in depth? Show it for tenant isolation in Beacon.
5. Why does row-level security need to fail closed, and how does the policy in section 4 do that?
6. Where does tenant isolation most often fail outside the main request path?
7. Describe three supply-chain attacks and the control that would have reduced each one's impact.
8. Why pin GitHub Actions by SHA? Why is `pull_request_target` dangerous?
9. What is an SBOM, and when does it pay off?
10. What should you be able to revoke quickly, and how?

**Exercises**

1. Threat model a Beacon feature with STRIDE: attachments upload and download (think about file types,
   size, malware, content sniffing, direct Blob URLs, cross-tenant access, and AI processing of
   attachments).
2. Implement row-level security for Beacon's tenant-owned tables and a cross-tenant test suite generated
   from the endpoint list.
3. Harden Beacon's GitHub Actions workflows: pin actions by SHA, set minimal permissions, add CodeQL,
   dependency review, secret scanning push protection and artifact attestations.
4. Generate an SBOM for the `beacon-api` image and write the query that answers "which images contain
   package X at version Y?"
5. Write a runbook for "a webhook secret was posted in a public channel" and time a dry run.

**Interview-style questions**

- "How would you design tenant isolation for a multi-tenant SaaS database?"
- "Walk me through how you'd threat model a new feature."
- "What would you do if a critical vulnerability was announced in a library you use?"
- "How do you secure a CI/CD pipeline?"
- "What's the difference between authentication and authorization failures? Give an example of each."

---

## 12. Going deeper

- Adam Shostack, *Threat Modeling: Designing for Security* and *Threats: What Every Engineer Should Learn
  from Star Wars*.
- [OWASP Top 10](https://owasp.org/Top10/), [OWASP ASVS](https://owasp.org/www-project-application-security-verification-standard/)
  (a detailed verification checklist) and the [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org).
- [SLSA framework](https://slsa.dev) and [OpenSSF Scorecard](https://securityscorecards.dev) for supply
  chain security.
- [GitHub Actions security hardening guide](https://docs.github.com/actions/security-for-github-actions/security-guides/security-hardening-for-github-actions).
- Microsoft, [Security Development Lifecycle](https://www.microsoft.com/securityengineering/sdl) and
  [ASP.NET Core security documentation](https://learn.microsoft.com/aspnet/core/security/).
- Ross Anderson, *Security Engineering* (3rd ed., free chapters online) — the classic, broad reference.

**Next:** [Maintainability, Technical Debt and Legacy Modernization](07-maintainability-technical-debt-and-legacy-modernization.md)
looks at how systems age, and how to keep changing them safely when they do.
