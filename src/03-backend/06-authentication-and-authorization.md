# Authentication and Authorization

Security mistakes in authentication and authorization are among the most damaging bugs
an application can have. **Broken access control** has topped the OWASP Top 10 list of web
application risks for years: users reading other users' data by changing an ID in the
URL, admin endpoints anyone can call, tokens that never expire.

The terminology is also notoriously confusing: OAuth, OpenID Connect, JWT, bearer tokens,
cookies, claims, scopes, roles, policies. This chapter untangles them, shows how they fit
together, and secures Beacon's API.

---

## 1. The problem: two different questions

Every protected request must answer two distinct questions:

- **Authentication (AuthN)**: *Who are you?* Verifying identity.
- **Authorization (AuthZ)**: *What are you allowed to do?* Checking permissions for this
  specific action on this specific resource.

They fail differently, and HTTP reflects that: **401 Unauthorized** means "not
authenticated" (despite the name); **403 Forbidden** means "authenticated, but not
allowed."

> **🧱 Durable:** Authentication is a solved problem you should **not** build yourself:
> use a proven identity provider and standard protocols. Authorization is specific to
> your domain, and you **must** design it yourself, carefully, because no library knows
> that "agents can only see tickets for their team."

---

## 2. The mental model: identities, credentials, tokens and claims

### From login to request

```text
 1. User proves identity to an Identity Provider (password + MFA, passkey, SSO)
 2. Identity Provider issues a token: a signed statement "this is maria; she's an agent; valid for 1 hour"
 3. Client sends the token with each API request
 4. API verifies the token's signature and validity → builds a ClaimsPrincipal
 5. API checks authorization rules against the principal's claims and the resource
```

### Claims

A **claim** is a statement about the subject: `sub` (subject ID) = `8f3a...`,
`name` = `Maria Lopez`, `email` = `maria@example.com`, `role` = `agent`,
`team` = `network`. In .NET, the authenticated user is a `ClaimsPrincipal`
(`HttpContext.User`) holding one or more `ClaimsIdentity`s with their claims.

### Sessions vs tokens

Two broad ways a client proves identity on each request:

| | Cookie-based session | Bearer token (e.g. JWT) |
|---|---|---|
| How it travels | Browser sends a cookie automatically | Client adds `Authorization: Bearer <token>` |
| Where state lives | Server-side session, or an encrypted cookie | In the token itself (self-contained) |
| Revocation | Easy: delete the session | Hard: valid until it expires |
| CSRF risk | Yes (cookies are sent automatically) | No (unless stored in a cookie) |
| XSS risk | Low with `HttpOnly` cookies | High if stored in JavaScript-accessible storage |
| Best for | Browser apps talking to their own backend | APIs called by other services, mobile apps, SPAs via a BFF |

Book VI, Chapter 5 covers the browser side in depth; the short version is that for browser
apps, the **BFF (Backend-for-Frontend)** pattern with `HttpOnly` cookies is generally the
most secure approach, while bearer tokens are natural for service-to-service and mobile
clients.

---

## 3. JWT: JSON Web Tokens

A JWT is a compact, signed token with three base64url-encoded parts:

```text
eyJhbGciOiJSUzI1NiIsImtpZCI6ImFiYzEyMyJ9 . eyJzdWIiOiI4ZjNhIiwiYXVkIjoiYmVhY29uLWFwaSIsInJvbGUiOiJhZ2VudCIsImV4cCI6MTc5MDAwMDAwMH0 . <signature>
        header                                                payload (claims)                                                   signature
```

Decoded:

```json
// header
{ "alg": "RS256", "kid": "abc123", "typ": "JWT" }

// payload
{
  "iss": "https://login.example.com/",     // issuer: who issued it
  "aud": "beacon-api",                     // audience: who it's for
  "sub": "8f3a2c1e-...",                   // subject: who it's about
  "name": "Maria Lopez",
  "role": "agent",
  "team": "network",
  "scp": "tickets.read tickets.write",     // scopes (delegated permissions)
  "iat": 1790000000,                       // issued at
  "exp": 1790003600                        // expires (1 hour later)
}
```

Key facts:

- **Signed, not encrypted.** Anyone can decode and read a JWT. Never put secrets or
  sensitive personal data in it.
- **The signature proves integrity and origin.** With **RS256/ES256** (asymmetric), the
  identity provider signs with a private key, and APIs verify with the public key, which
  they download from the provider's JWKS endpoint (identified by `kid`). With **HS256**
  (symmetric), the same secret signs and verifies, so every verifier could also forge tokens.
- **Self-contained**: the API doesn't call the identity provider per request, which makes
  validation fast. The flip side: a stolen token is valid until `exp`. Keep access tokens
  **short-lived** (minutes to an hour) and use **refresh tokens** to get new ones.

What an API must validate (ASP.NET Core's JWT handler does all of it when configured):

1. Signature, with a trusted key.
2. `iss` matches the expected issuer.
3. `aud` matches *this* API (a token for another API must be rejected).
4. `exp` / `nbf`: not expired, already valid (with small clock skew).
5. The algorithm is the expected one (never accept `alg: none`).

---

## 4. OAuth 2.0 and OpenID Connect

These two protocols are where most confusion lives.

- **OAuth 2.0** is a framework for **delegated authorization**: letting an application
  access an API *on behalf of* a user (or itself) without handling the user's password.
  It issues **access tokens**. Example: "Let this calendar app read my Google Calendar."
- **OpenID Connect (OIDC)** is a layer on top of OAuth 2.0 for **authentication**: it adds
  an **ID token** (a JWT about the user) and a standard user-info endpoint. Example:
  "Sign in with Google/Microsoft."

### Roles in OAuth

| Role | In Beacon's world |
|---|---|
| Resource owner | Maria, the user |
| Client | Beacon's React frontend (or its BFF), or a partner integration |
| Authorization server | The identity provider (Microsoft Entra ID, Auth0, Keycloak, Duende IdentityServer) |
| Resource server | Beacon.Api |

### The flows you'll actually use

| Flow | Use for |
|---|---|
| **Authorization Code + PKCE** | Users signing in through a browser or mobile app. The standard for every interactive client. |
| **Client Credentials** | Service-to-service calls with no user (a background job calling Beacon.Api). |
| **Refresh Token** | Getting a new access token without making the user sign in again. |
| **Device Code** | Devices without a browser (CLIs, TVs). |

Obsolete flows you should not use: **Implicit** (tokens in URL fragments) and **Resource
Owner Password Credentials** (the app collects the user's password).

### Authorization Code + PKCE, briefly

```text
 Browser/App                       Identity Provider                     Beacon.Api
 ───────────                       ─────────────────                     ──────────
 1. redirect to /authorize ──────► user signs in (MFA, SSO...)
    (client_id, redirect_uri,
     scope, code_challenge)
 2. ◄───── redirect back with a one-time code
 3. POST /token (code + code_verifier) ──► verifies PKCE, returns
                                           access_token (+ id_token, refresh_token)
 4. GET /api/tickets  Authorization: Bearer <access_token> ──────────────────► validates token
```

**PKCE** (Proof Key for Code Exchange) proves that the party exchanging the code is the
same one that started the flow, which protects public clients (SPAs, mobile apps) that
can't keep a client secret.

### Scopes vs roles

- **Scopes** (`tickets.read`) describe what a *client application* is allowed to do on the
  user's behalf. They limit delegation.
- **Roles / permissions** (`agent`, `lead`) describe what the *user* is allowed to do.

An action should require both: the user must have permission, *and* the client must have
been granted a scope covering it.

> **🧭 When not to build your own identity provider:** Almost always. Password storage,
> MFA, account recovery, brute-force protection, passkeys, token issuance and key
> rotation are specialized, high-risk work. Use a hosted provider (Microsoft Entra ID,
> Entra External ID for customers, Auth0, Okta) or a well-maintained product
> (Keycloak, Duende IdentityServer). ASP.NET Core Identity is a reasonable choice for
> a self-contained app with its own user database and cookie authentication.

---

## 5. Authentication in ASP.NET Core

The framework separates **authentication handlers** (schemes) from **authorization**:

```csharp
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://login.example.com/";   // discovers keys via /.well-known/openid-configuration
        options.Audience = "beacon-api";
        options.MapInboundClaims = false;                     // keep claim names as in the token ("sub", "role")
        options.TokenValidationParameters.RoleClaimType = "role";
        options.TokenValidationParameters.NameClaimType = "name";
    });

builder.Services.AddAuthorization();

var app = builder.Build();
app.UseAuthentication();     // builds HttpContext.User from the token
app.UseAuthorization();      // enforces policies on endpoints
```

With `Authority` set, the handler downloads the provider's signing keys and refreshes them
automatically when keys rotate.

`MapInboundClaims = false` is worth calling out: by default, the handler renames standard
claims to long legacy URIs (`sub` becomes
`http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier`), which confuses
everyone. Turning it off keeps claim names as they appear in the token.

### Local development tokens

For development, `dotnet user-jwts` creates a local signing key and issues test tokens:

```bash
dotnet user-jwts create --project src/Beacon.Api --name maria --role agent --claim team=network
```

It prints a token and configures `appsettings.Development.json` so the JWT handler trusts
it. In production, the configuration points at the real identity provider.

---

## 6. Authorization in ASP.NET Core

### Requiring authentication

```csharp
var tickets = app.MapGroup("/api/tickets").RequireAuthorization();   // all endpoints need a user
app.MapGet("/health", ...).AllowAnonymous();
```

Even better, make authentication the default and opt *out* explicitly:

```csharp
builder.Services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder().RequireAuthenticatedUser().Build());
```

Now any endpoint someone forgets to protect is protected anyway. **Secure by default.**

### Policies

A **policy** is a named set of requirements:

```csharp
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("Agent", p => p.RequireRole("agent", "lead"))
    .AddPolicy("Lead", p => p.RequireRole("lead"))
    .AddPolicy("TicketsWrite", p => p.RequireClaim("scp", "tickets.write"));   // simplified; scopes are space-separated

group.MapPost("/{id}/resolve", Resolve).RequireAuthorization("Agent");
```

Roles work for coarse checks. Prefer policies named after **capabilities** ("CanResolve")
rather than roles at the endpoint, so changing who has the capability changes one place.

### Resource-based authorization

Role checks answer "can agents resolve tickets?" They can't answer "can *this* agent
resolve *this* ticket?" (Only tickets for their team.) That needs the resource:

```csharp
public sealed class TicketAuthorizationHandler
    : AuthorizationHandler<OperationAuthorizationRequirement, Ticket>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context, OperationAuthorizationRequirement requirement, Ticket ticket)
    {
        var user = context.User;
        var isLead = user.IsInRole("lead");
        var sameTeam = user.FindFirst("team")?.Value == ticket.Team;

        var allowed = requirement.Name switch
        {
            "Read"    => isLead || sameTeam || ticket.ReporterId == user.FindFirst("sub")?.Value,
            "Resolve" => isLead || (user.IsInRole("agent") && sameTeam),
            _         => false,
        };

        if (allowed) context.Succeed(requirement);
        return Task.CompletedTask;
    }
}
```

```csharp
var auth = await authorizationService.AuthorizeAsync(user, ticket, new OperationAuthorizationRequirement { Name = "Resolve" });
if (!auth.Succeeded) return TypedResults.Forbid();
```

### IDOR: the most common access-control bug

**Insecure Direct Object Reference**: an endpoint loads a resource by ID from the URL and
returns it without checking that the caller may see it.

```csharp
// ✗ Any authenticated user can read any ticket by guessing IDs
app.MapGet("/api/tickets/{id}", async (TicketId id, ITicketRepository repo) => await repo.FindAsync(id));
```

Every endpoint that takes a resource ID must authorize access to *that resource*. Two
reliable techniques:

1. **Resource-based authorization** after loading (above).
2. **Scope the query by the user**: `WHERE id = @id AND team_id = @userTeam`. The
   resource simply isn't found for users who can't see it. Returning **404 rather than
   403** also avoids revealing that the resource exists.

> **⚠️ What can go wrong:** Authorization checks in the frontend only (hiding a button)
> protect nothing. The API is directly callable by anyone with a token. Every rule must be
> enforced server-side.

---

## 7. Other credentials you'll meet

- **API keys**: long random strings identifying a calling *application*. Simple for
  server-to-server integrations. They're not user identity, are often long-lived, and
  leak easily. Hash them at rest, allow rotation, scope them, and prefer OAuth client
  credentials when possible.
- **Managed identities / workload identity**: cloud-issued identities for your services, so
  your API can call Azure SQL, Key Vault or another API with no secret at all (Book IX).
- **mTLS**: both client and server present certificates. Common inside service meshes
  and for high-security B2B integrations.
- **Passkeys (WebAuthn/FIDO2)**: phishing-resistant, passwordless sign-in using device
  keys. Increasingly the recommended user authentication method; handled by your identity
  provider.

---

## 8. In practice: securing Beacon.Api

Beacon has three kinds of users:

| Role | Can |
|---|---|
| `customer` | Create tickets; read and comment on tickets they reported |
| `agent` | Read and work on tickets for their team; assign, comment, resolve |
| `lead` | Everything agents can, for all teams; reopen and close |

The domain gains two properties on `Ticket`: `ReporterId` (the `sub` of the user who
created it) and `Team`.

### Authentication and policies

```csharp
// Program.cs (excerpt)
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(o =>
    {
        // Authority and Audience come from configuration (Chapter 3); user-jwts fills them in Development.
        o.MapInboundClaims = false;
        o.TokenValidationParameters.RoleClaimType = "role";
        o.TokenValidationParameters.NameClaimType = "name";
    });

builder.Services.AddAuthorizationBuilder()
    .SetFallbackPolicy(new AuthorizationPolicyBuilder().RequireAuthenticatedUser().Build())
    .AddPolicy(Policies.Staff, p => p.RequireRole(Roles.Agent, Roles.Lead))
    .AddPolicy(Policies.Lead, p => p.RequireRole(Roles.Lead));

builder.Services.AddSingleton<IAuthorizationHandler, TicketAuthorizationHandler>();
```

```csharp
// src/Beacon.Api/Security/SecurityConstants.cs
namespace Beacon.Api.Security;

public static class Roles { public const string Customer = "customer", Agent = "agent", Lead = "lead"; }
public static class Policies { public const string Staff = "Staff", Lead = "Lead"; }
public static class TicketOperations
{
    public static readonly OperationAuthorizationRequirement Read = new() { Name = "Read" };
    public static readonly OperationAuthorizationRequirement Work = new() { Name = "Work" };
}
```

### A current-user abstraction

The core shouldn't know about `HttpContext`, but services need "who is acting":

```csharp
// src/Beacon.Core/Common/ICurrentUser.cs
namespace Beacon.Core.Common;

public interface ICurrentUser
{
    string Id { get; }
    string DisplayName { get; }
}
```

```csharp
// src/Beacon.Api/Security/HttpCurrentUser.cs
using System.Security.Claims;
using Beacon.Core.Common;

namespace Beacon.Api.Security;

public sealed class HttpCurrentUser(IHttpContextAccessor accessor) : ICurrentUser
{
    private ClaimsPrincipal User => accessor.HttpContext?.User
        ?? throw new InvalidOperationException("No HTTP context.");

    public string Id => User.FindFirstValue("sub") ?? throw new InvalidOperationException("Token has no 'sub'.");
    public string DisplayName => User.FindFirstValue("name") ?? Id;
}
```

Registered as scoped (`AddHttpContextAccessor()`, `AddScoped<ICurrentUser, HttpCurrentUser>()`).
Now comments use the authenticated user as the author, and the `Author` field disappears
from `AddCommentRequest`: a client can no longer comment as someone else. That's
over-posting prevention (Chapter 5) and authentication working together.

### Endpoints with resource checks

```csharp
private static async Task<IResult> Get(
    TicketId id, ITicketRepository repo, IAuthorizationService authz, ClaimsPrincipal user, CancellationToken ct)
{
    var ticket = await repo.FindAsync(id, ct);
    if (ticket is null) return TypedResults.NotFound();

    var result = await authz.AuthorizeAsync(user, ticket, TicketOperations.Read);
    return result.Succeeded
        ? TypedResults.Ok(TicketResponse.From(ticket))
        : TypedResults.NotFound();            // don't reveal that the ticket exists
}
```

The list endpoint filters by what the user may see (customers: their own; agents: their
team; leads: all) *in the query*, rather than loading everything and filtering, which
becomes a `WHERE` clause in Book IV.

### Testing authorization

Authorization rules are exactly the kind of logic that deserves thorough tests. With
`WebApplicationFactory`, replace authentication with a test scheme that builds a principal
from request headers, then test the matrix:

| Caller | Ticket | Operation | Expected |
|---|---|---|---|
| Customer (reporter) | own | Read | 200 |
| Customer (not reporter) | other's | Read | 404 |
| Agent, same team | team's | Resolve | 200 |
| Agent, other team | other team's | Resolve | 404 |
| Lead | any | Reopen | 200 |
| No token | any | any | 401 |

A `[Theory]` over this table is one of the highest-value tests in the whole suite.

---

## 9. What can go wrong

- **Missing authorization on one endpoint** (fix: fallback policy).
- **IDOR**: checking authentication but not ownership.
- **Trusting client-supplied identity**: `author` or `userId` fields in requests.
- **Long-lived tokens** with no revocation path.
- **Not validating `aud`**: accepting tokens meant for other APIs.
- **Symmetric signing keys shared widely**, or secrets committed to configuration.
- **Sensitive data in JWTs**, which anyone can decode.
- **Tokens in URLs** (logged by proxies and browsers) or in `localStorage` (readable by any
  XSS).
- **Role explosion**: dozens of roles checked ad hoc in endpoints. Use policies and
  capabilities.
- **Leaking existence** with 403 where 404 would be safer.

---

## 10. How an experienced engineer thinks about this

- **Buy authentication, design authorization.**
- **Deny by default.** Fallback policies, explicit `AllowAnonymous`.
- **Authorize every resource access**, not just every endpoint.
- **Identity comes from the token, never the request body.**
- **Short-lived credentials, least privilege, easy rotation.**
- **Test the authorization matrix** explicitly; it's where the costliest bugs hide.

---

## 11. Check yourself

**Questions**

1. What's the difference between authentication and authorization? Between 401 and 403?
2. What are the three parts of a JWT? Is the payload secret?
3. What must an API validate in an incoming JWT?
4. What's the difference between OAuth 2.0 and OpenID Connect?
5. Which OAuth flow should a SPA use? A background service?
6. What's IDOR, and two ways to prevent it?
7. Why use a fallback authorization policy?

**Exercises**

1. Use `dotnet user-jwts` to create tokens for a customer, an agent and a lead, and call
   Beacon's endpoints with each.
2. Implement the authorization matrix test from section 8 with a test authentication handler.
3. Decode a JWT at a site like jwt.io (with a test token only) and identify every claim.
4. Add an API-key scheme for a partner integration that can only create tickets, with keys
   stored hashed.

**Interview-style questions**

- "Explain OAuth 2.0 Authorization Code flow with PKCE."
- "How does JWT validation work? What are the risks of JWTs?"
- "How would you make sure users can only access their own data?"
- "Cookies or tokens for a single-page application? Why?"

---

## 12. Going deeper

- [Microsoft docs: Authentication in ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/authentication/)
- [Microsoft docs: Authorization in ASP.NET Core](https://learn.microsoft.com/aspnet/core/security/authorization/introduction)
- [oauth.net](https://oauth.net/2/) and the OAuth 2.0 Security Best Current Practice (RFC 9700).
- [OWASP Authorization Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authorization_Cheat_Sheet.html)
- Aaron Parecki, *OAuth 2.0 Simplified*.

**Next:** [Chapter 7 — Caching](07-caching.md) makes Beacon faster without making it wrong.
