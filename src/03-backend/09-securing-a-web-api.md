# Securing a Web API

Chapter 6 covered *who* can call Beacon and *what* they may do. That's necessary but
not sufficient. A perfectly authenticated API can still leak data through SQL injection,
be taken down by a single client sending huge requests, be abused through a browser with
cross-site request forgery, or be compromised through a vulnerable NuGet package.

Security isn't a feature you add at the end; it's a property of every decision. This
chapter gives you a practical threat model and checklist for web APIs, organized around
the risks that actually cause incidents.

---

## 1. The problem: the API is reachable by attackers

Anything on the internet is probed constantly by automated scanners, within minutes of
going live. Attackers don't need to break your cryptography; they look for:

- input your code trusts but shouldn't,
- checks you forgot in one endpoint out of fifty,
- defaults nobody changed,
- dependencies with known vulnerabilities,
- secrets in places they shouldn't be,
- resources they can exhaust.

The defender has to get everything right; the attacker needs one gap. The defense is
**layers** (*defense in depth*) and **defaults that are safe** even when someone forgets
something.

---

## 2. The mental model: threat modeling

Before a checklist, a way of thinking. **Threat modeling** asks four questions (Adam
Shostack's framework):

1. **What are we building?** Draw the data flow: clients, API, database, third parties,
   trust boundaries.
2. **What can go wrong?** Walk each boundary with a prompt like **STRIDE**:

| Threat | Question | Beacon example |
|---|---|---|
| **S**poofing | Can someone pretend to be someone else? | Forged JWT; comment as another user |
| **T**ampering | Can someone modify data they shouldn't? | Over-posting `status`; editing others' tickets |
| **R**epudiation | Can someone deny doing something? | No audit trail of who closed a ticket |
| **I**nformation disclosure | Can someone see data they shouldn't? | IDOR on tickets; stack traces in errors |
| **D**enial of service | Can someone make it unavailable? | 1 GB request bodies; expensive search queries |
| **E**levation of privilege | Can someone gain more rights? | Customer calling lead-only endpoints |

3. **What are we going to do about it?** Mitigate, accept, or transfer each risk.
4. **Did we do a good job?** Review, test, revisit when the design changes.

A one-hour whiteboard session per major feature catches more than most scanners.

---

## 3. The OWASP Top 10, applied to APIs

The **OWASP Top 10** (and the more specific **OWASP API Security Top 10**) list the most
common and impactful risk categories. Here they are, mapped to concrete defenses in
ASP.NET Core.

### Broken access control (and BOLA/IDOR)

The #1 risk. Covered in Chapter 6: fallback policy, resource-based authorization on every
object access, scoping queries by the user, never trusting client-supplied identity.

### Injection

Untrusted input interpreted as code: SQL, OS commands, LDAP, and more.

```csharp
// ✗ SQL injection: title = "x'; DROP TABLE tickets; --"
var sql = $"SELECT * FROM tickets WHERE title = '{title}'";

// ✓ Parameterized: the value is data, never SQL
await using var cmd = new NpgsqlCommand("SELECT * FROM tickets WHERE title = @title", conn);
cmd.Parameters.AddWithValue("title", title);

// ✓ EF Core LINQ is parameterized automatically
db.Tickets.Where(t => t.Title == title);

// ✓ EF Core interpolated raw SQL is parameterized too (FromSql / SqlQuery)
db.Tickets.FromSql($"SELECT * FROM tickets WHERE title = {title}");

// ✗ ...but FromSqlRaw with string concatenation is not
db.Tickets.FromSqlRaw("SELECT * FROM tickets WHERE title = '" + title + "'");
```

Dynamic `ORDER BY` columns can't be parameterized; **map user input to an allow-list**:

```csharp
var orderBy = sort switch
{
    "priority" => "priority",
    "createdAt" => "created_at",
    _ => throw new ValidationException("Unsupported sort field."),
};
```

Never pass user input to `Process.Start` arguments, file paths (path traversal:
`../../etc/passwd`) or template engines without strict validation.

### Cryptographic failures

- **HTTPS everywhere**, with HSTS (`app.UseHsts()` in production) so browsers never
  downgrade.
- **Never invent cryptography.** Use the platform: `RandomNumberGenerator` for tokens
  (never `Random`), `Rfc2898DeriveBytes.Pbkdf2` or ASP.NET Core Identity's hasher for
  passwords (or better, don't store passwords; Chapter 6), `AesGcm` for encryption, Data
  Protection APIs for protecting cookies and short-lived payloads.
- **Encrypt sensitive data at rest** where required, and manage keys in a key vault.

### Insecure design

Flaws in the design itself, not the implementation: a password-reset flow that reveals
whether an email exists, an "export all" endpoint with no limits, business logic that lets
a coupon be applied twice. Threat modeling (section 2) is the defense.

### Security misconfiguration

- Developer exception page or detailed errors in production.
- Default credentials, open admin endpoints (`/metrics`, `/hangfire`, Swagger UI) exposed
  publicly.
- Overly permissive CORS.
- Verbose headers revealing versions (`Server: Kestrel`): remove with
  `options.AddServerHeader = false`.
- Debug logging enabled in production, logging secrets.

### Vulnerable and outdated components

Your API is mostly other people's code: the .NET runtime, ASP.NET Core and dozens of NuGet
packages. Defenses:

- **Patch the runtime** monthly (.NET Patch Tuesday). Container base images must be rebuilt
  to pick up patches.
- **Scan dependencies**: `dotnet list package --vulnerable --include-transitive`, NuGet
  audit warnings (on by default in recent SDKs), Dependabot or Renovate.
- **Minimize dependencies.** Every package is code you trust.
- **Lock and verify** package sources (`nuget.config` with package source mapping) to
  prevent dependency confusion attacks.

### Identification and authentication failures

Chapter 6: use a real identity provider, short-lived tokens, MFA/passkeys, rate-limit
login and reset endpoints.

### Software and data integrity failures

Unsafe deserialization (Chapter 5), unsigned updates, CI/CD pipelines that can be modified
by anyone, packages pulled from untrusted sources. Book IX covers pipeline security.

### Security logging and monitoring failures

If you can't see an attack, you can't respond. Log authentication failures, authorization
denials, validation failures at unusual rates and admin actions, with user IDs and trace
IDs (without secrets). Alert on anomalies (Book IX, Chapter 6).

### Server-side request forgery (SSRF)

When your server fetches a URL supplied by a user (webhooks, "import from URL", link
previews), an attacker can point it at internal addresses: `http://169.254.169.254/` (cloud
metadata service, which can hand out credentials), `http://localhost:6379` (Redis), internal
admin APIs. Defenses: allow-list destinations, resolve and block private/link-local IP
ranges, disable redirects or re-validate after them, and run such fetches from a network
segment with no access to internal services.

---

## 4. Browser-specific threats: CORS, CSRF and XSS

### CORS

Browsers enforce the **same-origin policy**: JavaScript on `https://app.example.com` can't
read responses from `https://api.example.com` unless the API allows it via **CORS** headers.

```csharp
builder.Services.AddCors(o => o.AddPolicy("Frontend", p => p
    .WithOrigins("https://app.beacon.example.com")      // explicit origins, never "*" with credentials
    .WithMethods("GET", "POST", "PATCH", "DELETE")
    .WithHeaders("Content-Type", "Authorization", "If-Match", "Idempotency-Key")
    .WithExposedHeaders("ETag", "Location")));

app.UseCors("Frontend");
```

Important: **CORS is not a security boundary for your API.** It only controls what
*browsers* let *JavaScript* read. curl, Postman and attackers' scripts ignore it entirely.
A misconfigured CORS policy (reflecting any origin with credentials) *weakens* security by
letting malicious sites read authenticated responses; a correct one doesn't protect you
from non-browser clients.

### CSRF

**Cross-site request forgery**: a malicious site causes the victim's browser to send a
request to your site, and the browser **automatically attaches the victim's cookies**.

- APIs authenticated with **bearer tokens in headers** aren't vulnerable (browsers don't
  attach them automatically).
- APIs or BFFs authenticated with **cookies** are, and need defenses: `SameSite=Lax` or
  `Strict` cookies, antiforgery tokens, and requiring a custom header (which forces a CORS
  preflight). Book VI, Chapter 5 implements this for the BFF.

### XSS

**Cross-site scripting** happens in the frontend (Book VI), but the API contributes:
storing user content (ticket titles, comments) and returning it. Defenses on the API side:
return JSON with the correct `Content-Type` (never HTML built from user input), set
`X-Content-Type-Options: nosniff`, and let the frontend encode output. If you accept rich
text (Markdown or HTML in comments), sanitize it with a vetted library on the server.

---

## 5. Resource exhaustion and abuse

Availability is a security property. Protect the API from both malicious and accidental
overload:

- **Request size limits**: Kestrel's default max body size (~28.6 MB) is too large for
  most JSON endpoints. Lower it globally, raise it per endpoint where needed
  (`[RequestSizeLimit]`, `.WithRequestTimeout`, `DisableRequestSizeLimit` for uploads).
- **Input limits**: maximum string lengths, array sizes, page sizes (Chapters 4 and 5).
- **Timeouts**: request timeouts middleware (`AddRequestTimeouts`), database command
  timeouts, `HttpClient` timeouts.
- **Rate limiting**: limit requests per user, per API key, or per IP:

```csharp
builder.Services.AddRateLimiter(o =>
{
    o.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    o.AddPolicy("per-user", http => RateLimitPartition.GetTokenBucketLimiter(
        http.User.FindFirst("sub")?.Value ?? http.Connection.RemoteIpAddress?.ToString() ?? "anon",
        _ => new TokenBucketRateLimiterOptions
        {
            TokenLimit = 100,                         // burst
            TokensPerPeriod = 50,                     // sustained
            ReplenishmentPeriod = TimeSpan.FromSeconds(10),
            QueueLimit = 0,
        }));
});

app.UseRateLimiter();
app.MapGroup("/api").RequireRateLimiting("per-user");
```

  Stricter limits for expensive or sensitive endpoints (login, password reset, search,
  exports). In-process limits are per instance; for global limits, use a gateway or a
  Redis-backed limiter.

- **Expensive queries**: full-text search, reports and exports can be abused. Limit them,
  queue them, or cache them.

---

## 6. Security headers

For APIs that only return JSON, the important headers are few:

```csharp
app.Use(async (ctx, next) =>
{
    ctx.Response.OnStarting(() =>
    {
        var h = ctx.Response.Headers;
        h["X-Content-Type-Options"] = "nosniff";
        h["Referrer-Policy"] = "no-referrer";
        h["Content-Security-Policy"] = "default-src 'none'; frame-ancestors 'none'";   // JSON needs nothing
        return Task.CompletedTask;
    });
    await next(ctx);
});
app.UseHsts();   // Strict-Transport-Security, production only
```

Frontends need a carefully designed Content Security Policy (Book VI).

---

## 7. Secrets and data protection

Chapter 3 covered configuration secrets. A few more rules:

- **Never log tokens, passwords, API keys** or full request bodies on auth endpoints.
- **Hash API keys** like passwords; show them to the user only once.
- **Minimize personal data**: collect less, retain it for less time, restrict who can read
  it, and know where it flows (logs, analytics, AI providers in Book XI).
- **Use ASP.NET Core Data Protection** (with keys persisted to a shared store and protected
  in production) for anything the framework encrypts: cookies, antiforgery tokens.

---

## 8. In practice: a security review of Beacon.Api

Here's the review, organized as the checklist you'd run on any API. Items marked ✓ were
handled in earlier chapters; items marked ➕ are added now.

**Authentication and authorization**
- ✓ JWT bearer with issuer, audience, signature, lifetime validation (Chapter 6).
- ✓ Fallback policy requiring authentication.
- ✓ Resource-based authorization on every ticket access, including SignalR (Chapters 6, 8).
- ✓ Author comes from the token, not the request body.

**Input handling**
- ✓ Typed contracts, unknown fields rejected, string length limits (Chapter 5).
- ✓ Page size capped; sort fields allow-listed (Chapter 4).
- ➕ Body size limit lowered globally:

```csharp
builder.WebHost.ConfigureKestrel(k =>
{
    k.Limits.MaxRequestBodySize = 1 * 1024 * 1024;   // 1 MB for JSON APIs
    k.AddServerHeader = false;
});
```

**Abuse**
- ➕ Per-user token-bucket rate limiting (section 5); a stricter policy for ticket creation.
- ➕ Request timeouts:

```csharp
builder.Services.AddRequestTimeouts(o => o.DefaultPolicy = new() { Timeout = TimeSpan.FromSeconds(30) });
app.UseRequestTimeouts();
```

**Errors and information disclosure**
- ✓ Problem details without stack traces in production (Chapter 2).
- ✓ 404 instead of 403 for resources the user can't see.
- ➕ `Server` header removed; security headers added (section 6).

**Browser**
- ➕ CORS restricted to the frontend origin from configuration.
- (CSRF handled in the BFF, Book VI.)

**Dependencies and supply chain**
- ➕ CI step: `dotnet list package --vulnerable --include-transitive` fails the build on
  high-severity findings; Dependabot enabled for NuGet and GitHub Actions.

**Secrets and logging**
- ✓ No secrets in `appsettings*.json`; user secrets locally (Chapter 3).
- ✓ Logs carry IDs, not ticket content or tokens.

**Monitoring**
- ➕ Log authorization failures and rate-limit rejections at `Warning` with user ID and
  trace ID, so spikes can be alerted on (Book IX).

A last, often skipped step: **test the security properties**. Integration tests that
assert `401` without a token, `404` for another team's ticket, `429` after the limit,
`413` for an oversized body, and no `Server` header. Security regressions are regressions
like any other.

---

## 9. What can go wrong

- Assuming authentication means security.
- Building SQL (or commands, or paths) from strings.
- CORS `AllowAnyOrigin` with credentials, or treating CORS as access control.
- Detailed errors and debug endpoints exposed in production.
- No limits: body size, page size, rate, time.
- Outdated runtime and packages; unrebuilt container images.
- SSRF through "fetch this URL" features.
- Secrets and personal data in logs.
- Security reviewed once at launch, never again.

---

## 10. How an experienced engineer thinks about this

- **Think like an attacker** for every new endpoint: what if every input is hostile?
- **Defense in depth**: validation, authorization, parameterization, limits, monitoring;
  each catches what another misses.
- **Secure defaults** beat secure intentions: fallback policies, global limits, safe
  serializer settings.
- **Least privilege everywhere**: users, services, database accounts, CI tokens.
- **Security is continuous**: patching, scanning, reviewing and monitoring are routine
  work, not a project.

---

## 11. Check yourself

**Questions**

1. Walk through STRIDE for a "reset password" feature.
2. How do you prevent SQL injection with EF Core? When does EF Core *not* protect you?
3. Why isn't CORS a security boundary for your API?
4. Which APIs are vulnerable to CSRF, and why?
5. What is SSRF, and why is the cloud metadata endpoint a common target?
6. What limits should every public API enforce?
7. How do you keep dependencies secure over time?

**Exercises**

1. Draw Beacon's data-flow diagram (browser, API, database, identity provider, SMTP) and
   run STRIDE on each trust boundary.
2. Write integration tests for: no token → 401, oversized body → 413, rate limit → 429,
   other team's ticket → 404.
3. Run `dotnet list package --vulnerable --include-transitive` on a real project and
   triage the results.
4. Implement a "fetch link preview" endpoint safely, blocking private and link-local IPs
   after DNS resolution.

**Interview-style questions**

- "How would you secure a REST API?"
- "Explain SQL injection and how to prevent it."
- "What's the difference between CORS and CSRF?"
- "How do you protect an API from abuse?"

---

## 12. Going deeper

- [OWASP Top 10](https://owasp.org/Top10/) and [OWASP API Security Top 10](https://owasp.org/API-Security/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/) — practical guidance per topic.
- [Microsoft docs: ASP.NET Core security](https://learn.microsoft.com/aspnet/core/security/)
- Adam Shostack, *Threat Modeling: Designing for Security*.

**Next:** [Chapter 10 — Diagnosing API Failures](10-diagnosing-api-failures.md) is a
practical playbook for when things go wrong in production.
