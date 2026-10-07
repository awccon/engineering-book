# Authentication in the Browser

Book III, Chapter 6 secured Beacon.Api with JWT bearer tokens. Now the React app needs to
sign users in and call that API. This is where many single-page applications make
serious security mistakes: access tokens stored in `localStorage` where any injected
script can steal them, long-lived tokens with no revocation, refresh tokens exposed to
JavaScript, and cookies without CSRF protection.

The browser is a hostile environment: it runs code from many sources (your bundle, its
dependencies, browser extensions, possibly injected scripts). This chapter explains the
threats, compares the approaches, and implements the one current security guidance
recommends for apps like Beacon: the **Backend-for-Frontend (BFF)** pattern.

---

## 1. The problem: secrets in a hostile environment

To call the API on the user's behalf, the browser app needs some credential. Wherever that
credential is, two attacks dominate:

- **XSS (cross-site scripting)**: an attacker gets JavaScript running in your page (a
  vulnerable dependency, unsanitized user content, a compromised CDN script). That script
  can read anything JavaScript can read: `localStorage`, `sessionStorage`, in-memory
  variables, non-`HttpOnly` cookies. And it can make requests as the user.
- **CSRF (cross-site request forgery)**: a malicious site makes the user's browser send a
  request to your site, and the browser automatically attaches your site's cookies (Book
  III, Chapter 9).

Every browser authentication design is a trade-off between these two.

---

## 2. The mental model: three architectures

### Option A: tokens in the SPA (public client)

```text
 Browser (SPA) ──Authorization Code + PKCE──► Identity Provider
       │◄─────────── access token (+ refresh token) ─────┘
       │
       └── Authorization: Bearer <access token> ──► Beacon.Api
```

The SPA runs the OAuth flow itself (with a library like MSAL.js or oidc-client-ts), keeps
tokens in memory (or storage), and sends them to the API.

- ✓ Simple backend: the API only validates tokens.
- ✗ Tokens are accessible to any script that runs in the page. With XSS, an attacker can
  **exfiltrate tokens** and use them from their own machine until they expire.
- ✗ Refresh tokens in the browser are especially valuable to attackers; mitigations
  (rotation, short lifetimes, sender-constraining with DPoP) add complexity.

### Option B: Backend-for-Frontend (confidential client)

```text
 Browser (SPA) ──cookie (HttpOnly, Secure, SameSite)──► BFF (same origin as the SPA)
                                                        │  holds tokens server-side
                                                        ├──OIDC code flow──► Identity Provider
                                                        └──Bearer token────► Beacon.Api
```

A small server-side component, the **BFF**, on the same origin as the SPA, performs the
OAuth flow as a **confidential client** (with a client secret or certificate), stores the
tokens **server-side** (or in an encrypted cookie), and gives the browser only an
**`HttpOnly` session cookie**. API calls go through the BFF, which attaches the access token.

- ✓ **No tokens in the browser.** XSS can still make requests while the user's page is
  open (nothing fixes that), but it can't steal long-lived credentials.
- ✓ Refresh tokens never leave the server.
- ✓ Same-origin requests: no CORS configuration needed.
- ✗ A server component to run, and CSRF protection required (cookies are sent
  automatically).

### Option C: cookie sessions with a traditional server

The original web model: the server renders pages or serves the SPA and authenticates with
cookies directly (ASP.NET Core Identity or OIDC with cookie authentication). It's
effectively a BFF where the "BFF" is the whole application.

> **🧱 Durable:** For browser-based applications, current OAuth security guidance (the IETF
> "OAuth 2.0 for Browser-Based Applications" best current practice) recommends the BFF
> pattern as the most secure option, because it keeps tokens out of JavaScript entirely.
> Tokens in the browser are acceptable for lower-risk apps when a BFF isn't feasible, with
> short lifetimes, rotation and strong XSS defenses.

### Where to store a token (if you must)

| Storage | XSS can read it? | Survives refresh? | Notes |
|---|---|---|---|
| `localStorage` / `sessionStorage` | **Yes** | Yes / per tab | The most common and least safe choice |
| JavaScript memory | Yes (while running) | No | Better: shorter exposure; needs silent re-auth on reload |
| `HttpOnly` cookie | **No** | Yes | Requires CSRF protection; effectively the BFF/cookie model |

---

## 3. Cookies, done right

A session cookie for a BFF should look like:

```http
Set-Cookie: __Host-beacon-session=...; Path=/; Secure; HttpOnly; SameSite=Strict
```

| Attribute | Effect |
|---|---|
| `HttpOnly` | JavaScript can't read it (protects against token theft via XSS) |
| `Secure` | Sent only over HTTPS |
| `SameSite=Strict` / `Lax` | Not sent on cross-site requests (`Strict`), or only on top-level navigations (`Lax`); the main CSRF defense |
| `__Host-` prefix | Browser enforces `Secure`, `Path=/` and no `Domain` attribute, so subdomains can't set or overwrite it |
| `Path=/` | Sent to all paths |

### CSRF defense in depth

`SameSite` cookies stop most CSRF, but add a second layer for state-changing requests:

- Require a **custom header** (e.g. `X-CSRF: 1`) on all API calls. Browsers can't send
  custom headers cross-site without a CORS preflight, which your server won't approve. This
  is what the Duende BFF library does.
- Or use **antiforgery tokens** (synchronizer tokens).
- Never change state with `GET`.

---

## 4. XSS: the threat behind everything

Whatever architecture you choose, **preventing XSS** matters most, because XSS lets an
attacker act as the user in the page.

React helps a lot: JSX **escapes** values by default, so `{ticket.title}` containing
`<script>` renders as text. The remaining risks:

- **`dangerouslySetInnerHTML`** with untrusted content. If you render Markdown or rich text
  (comments, knowledge-base articles), sanitize it with a vetted library (DOMPurify) after
  converting to HTML.
- **URLs from user data**: `<a href={userUrl}>` with `javascript:alert(1)`. Validate that
  URLs use `http:`/`https:`.
- **Third-party scripts** (analytics, chat widgets, tag managers) run with full page
  access. Each is a supply-chain risk.
- **Compromised npm dependencies** (Book V, Chapter 3).

### Content Security Policy

A **CSP** header tells the browser which sources of script, style, images and connections
are allowed, so even if an attacker injects markup, their script won't run:

```http
Content-Security-Policy: default-src 'self'; script-src 'self'; style-src 'self';
  img-src 'self' data: https:; connect-src 'self'; frame-ancestors 'none';
  base-uri 'self'; form-action 'self'; object-src 'none'
```

A Vite-built SPA served from the same origin works well with a strict CSP (no inline
scripts). Start with `Content-Security-Policy-Report-Only` to find violations before
enforcing.

---

## 5. Sessions, expiry and sign-out

- **Session lifetime**: sliding expiration (e.g. 8 hours, extended with activity) with an
  absolute maximum (e.g. 24 hours).
- **Token refresh**: the BFF refreshes access tokens server-side before they expire; the
  browser never sees them.
- **Sign-out**: clear the BFF session, revoke the refresh token at the identity provider,
  and perform **front-channel or back-channel logout** so signing out of the identity
  provider ends the app session too.
- **401 handling in the SPA**: when the session expires, API calls return 401; the app
  should redirect to login (preserving where the user was) rather than showing errors.

---

## 6. In practice: a BFF for Beacon

Beacon adds a BFF to the existing ASP.NET Core host. Two options:

1. Add BFF endpoints into **Beacon.Api** itself (the SPA is served by the same app). Simple
   for one API.
2. A **separate `Beacon.Bff` project** that serves the SPA, handles login and proxies
   `/api/*` to Beacon.Api with the user's token. Cleaner separation when several APIs or
   clients exist.

Beacon uses option 2, because the API is also used by partner integrations and a future
mobile app with bearer tokens directly.

### The BFF host

```bash
dotnet new web -n Beacon.Bff -o src/Beacon.Bff
dotnet add src/Beacon.Bff package Microsoft.AspNetCore.Authentication.OpenIdConnect
dotnet add src/Beacon.Bff package Yarp.ReverseProxy
```

```csharp
// src/Beacon.Bff/Program.cs
using Microsoft.AspNetCore.Authentication;
using Microsoft.AspNetCore.Authentication.Cookies;
using Microsoft.AspNetCore.Authentication.OpenIdConnect;
using Yarp.ReverseProxy.Transforms;

var builder = WebApplication.CreateBuilder(args);

builder.Services
    .AddAuthentication(o =>
    {
        o.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
        o.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
    })
    .AddCookie(o =>
    {
        o.Cookie.Name = "__Host-beacon";
        o.Cookie.SameSite = SameSiteMode.Strict;
        o.Cookie.SecurePolicy = CookieSecurePolicy.Always;
        o.Cookie.HttpOnly = true;
        o.ExpireTimeSpan = TimeSpan.FromHours(8);
        o.SlidingExpiration = true;
        o.Events.OnRedirectToLogin = ctx =>               // API calls get 401, not an HTML redirect
        {
            if (ctx.Request.Path.StartsWithSegments("/api")) ctx.Response.StatusCode = 401;
            else ctx.Response.Redirect(ctx.RedirectUri);
            return Task.CompletedTask;
        };
    })
    .AddOpenIdConnect(o =>
    {
        builder.Configuration.Bind("Oidc", o);            // Authority, ClientId, ClientSecret from config/Key Vault
        o.ResponseType = "code";                           // Authorization Code flow (+ PKCE by default)
        o.SaveTokens = true;                               // tokens stored in the encrypted auth cookie (server-side store recommended at scale)
        o.Scope.Add("offline_access");
        o.Scope.Add("api://beacon-api/tickets");
        o.MapInboundClaims = false;
        o.GetClaimsFromUserInfoEndpoint = true;
    });

builder.Services.AddAuthorization();
builder.Services.AddReverseProxy()
    .LoadFromConfig(builder.Configuration.GetSection("ReverseProxy"))
    .AddTransforms(t => t.AddRequestTransform(async ctx =>
    {
        var token = await ctx.HttpContext.GetTokenAsync("access_token");   // refresh handling: see note below
        if (token is not null)
            ctx.ProxyRequest.Headers.Authorization = new("Bearer", token);
    }));

var app = builder.Build();

app.UseDefaultFiles();
app.UseStaticFiles();                                      // the built SPA (web/dist) in wwwroot
app.UseAuthentication();
app.UseAuthorization();

// Anti-CSRF: every API call from the SPA must carry X-CSRF: 1
app.Use(async (ctx, next) =>
{
    if (ctx.Request.Path.StartsWithSegments("/api") && ctx.Request.Headers["X-CSRF"] != "1")
    {
        ctx.Response.StatusCode = StatusCodes.Status400BadRequest;
        return;
    }
    await next(ctx);
});

app.MapGet("/bff/login", (string? returnUrl) =>
    Results.Challenge(new AuthenticationProperties { RedirectUri = SafeLocal(returnUrl) }));

app.MapPost("/bff/logout", () => Results.SignOut(
    new AuthenticationProperties { RedirectUri = "/" },
    [CookieAuthenticationDefaults.AuthenticationScheme, OpenIdConnectDefaults.AuthenticationScheme]))
   .RequireAuthorization();

app.MapGet("/bff/user", (HttpContext ctx) => ctx.User.Identity?.IsAuthenticated == true
    ? Results.Ok(new
      {
          id = ctx.User.FindFirst("sub")?.Value,
          name = ctx.User.FindFirst("name")?.Value,
          roles = ctx.User.FindAll("role").Select(c => c.Value),
      })
    : Results.Unauthorized());

app.MapReverseProxy().RequireAuthorization();              // /api/* and /hubs/* → Beacon.Api
app.MapFallbackToFile("index.html");                       // SPA fallback for client-side routes

app.Run();

static string SafeLocal(string? url) =>
    !string.IsNullOrEmpty(url) && url.StartsWith('/') && !url.StartsWith("//") ? url : "/";
```

Notes:

- **`SafeLocal`** prevents **open redirects** (`/bff/login?returnUrl=https://evil.example`).
- **Token refresh**: production BFFs refresh access tokens server-side before they expire.
  The **Duende.BFF** library (or `Duende.AccessTokenManagement`) implements this, plus
  server-side session storage, back-channel logout and CSRF handling, and is worth using
  rather than hand-rolling. The code above shows the moving parts so you understand what
  such a library does.
- **The SignalR hub** is proxied too (YARP supports WebSockets), so live updates use the same
  cookie session.

### The SPA side

The React app never sees a token. The API client from Book V adds the CSRF header and
handles 401 by redirecting to login:

```ts
// web/src/api/client.ts (changes)
headers: {
  Accept: 'application/json',
  'X-CSRF': '1',
  // ...
},
// ...
if (response.status === 401) {
  window.location.assign(`/bff/login?returnUrl=${encodeURIComponent(location.pathname + location.search)}`);
  return new Promise<never>(() => {});   // navigation in progress; never resolve
}
```

The session is loaded once at startup and provided through `SessionContext` (Chapter 3):

```tsx
// web/src/session/SessionGate.tsx
import { useQuery } from '@tanstack/react-query';

export function SessionGate({ children }: { children: React.ReactNode }) {
  const { data, status } = useQuery({
    queryKey: ['me'],
    queryFn: async () => {
      const r = await fetch('/bff/user', { headers: { 'X-CSRF': '1' } });
      if (r.status === 401) return null;
      if (!r.ok) throw new Error(`Failed to load session (${r.status})`);
      return (await r.json()) as BffUser;
    },
    staleTime: Infinity,
  });

  if (status === 'pending') return <FullPageSpinner />;
  if (status === 'error') return <FullPageError />;
  if (data === null) {
    window.location.assign(`/bff/login?returnUrl=${encodeURIComponent(location.pathname)}`);
    return null;
  }
  return <SessionProvider session={toSession(data)}>{children}</SessionProvider>;
}
```

### What we've gained

- No access or refresh tokens in JavaScript, storage or the URL.
- Cookies are `HttpOnly`, `Secure`, `SameSite=Strict`, `__Host-` prefixed.
- CSRF requires both a cross-site cookie (blocked by `SameSite`) and a custom header
  (blocked by CORS): two independent layers.
- The API remains a pure bearer-token resource server, usable by other clients.
- Same origin for SPA, API and hub: no CORS configuration in production.

Add a strict CSP to the BFF's responses (section 4), and the browser side of Beacon's
security is in good shape. Book IX hosts the BFF.

---

## 7. What can go wrong

- **Tokens in `localStorage`**, stolen by any XSS.
- **Refresh tokens in the browser** without rotation or constraints.
- **Cookies without `HttpOnly`, `Secure` or `SameSite`.**
- **No CSRF protection** for cookie-authenticated APIs.
- **Open redirects** in login/return URL handling.
- **`dangerouslySetInnerHTML` with unsanitized content**; unvalidated `href`s.
- **Third-party scripts** with full page access.
- **Treating hidden UI as authorization** (Chapter 3).
- **Session expiry handled as an error** instead of a re-login flow.

---

## 8. How an experienced engineer thinks about this

- **Assume XSS will happen eventually**, and design so it can't steal long-lived credentials.
- **Keep tokens server-side** for browser apps (BFF), unless there's a strong reason not to.
- **Defense in depth**: `HttpOnly`, `SameSite`, custom headers, CSP, sanitization.
- **Use well-maintained libraries** for OIDC and BFF mechanics; understand what they do.
- **The API enforces security; the UI reflects it.**

---

## 9. Check yourself

**Questions**

1. What can an XSS attacker do with a token in `localStorage` that they can't do with an
   `HttpOnly` cookie?
2. Describe the BFF pattern. Why is it a confidential client?
3. What do `HttpOnly`, `Secure`, `SameSite` and the `__Host-` prefix each do?
4. Why does a cookie-authenticated API need CSRF protection? How does a custom header help?
5. How does React protect against XSS by default? Where can XSS still get in?
6. What is a Content Security Policy?
7. What's an open redirect, and how do you prevent it?

**Exercises**

1. Run Beacon.Bff locally against a development identity provider (Keycloak in Docker, or a
   free Entra ID tenant), sign in, and confirm in DevTools that no tokens appear in
   JavaScript-accessible storage.
2. Try a CSRF attack from a page on another origin (a form posting to `/api/tickets`) and
   confirm it's blocked, and why.
3. Add a CSP in report-only mode and fix any violations.
4. Render a knowledge-base article's Markdown safely with a Markdown library plus DOMPurify.

**Interview-style questions**

- "Where should a single-page application store access tokens?"
- "What's the BFF pattern and why use it?"
- "How do you protect a cookie-based API from CSRF?"
- "How do you prevent XSS in a React application?"

---

## 10. Going deeper

- IETF, *OAuth 2.0 for Browser-Based Applications* (best current practice draft/RFC).
- [OWASP: Cross Site Scripting Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
  and [CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [Duende BFF documentation](https://docs.duendesoftware.com/bff/)
- [MDN: Content Security Policy](https://developer.mozilla.org/docs/Web/HTTP/CSP)

**Next:** [Chapter 6 — Performance and Accessibility](06-performance-and-accessibility.md)
makes Beacon fast and usable by everyone.
