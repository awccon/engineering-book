# HTTP from First Principles

Every backend developer works with HTTP daily, usually through layers of framework that
hide it: controllers, model binding, `HttpClient`. That's productive until something goes
wrong at the protocol level. A browser refuses a response because of CORS. A proxy caches
data it shouldn't. A client retries a `POST` and creates two orders. A request works in
Postman but not from the frontend. Diagnosing any of these requires knowing what's
actually on the wire.

This chapter covers HTTP itself: the message format, methods and their guarantees,
status codes, headers, caching, connections and the newer protocol versions. It's the
foundation for the rest of Book III.

---

## 1. The problem: a common language between strangers

The web connects programs written by people who never met, in different languages, on
different platforms, through intermediaries (proxies, CDNs, load balancers) that know
nothing about any of them. They need a protocol that is:

- **simple** enough to implement everywhere,
- **self-describing** so intermediaries can act on messages without understanding them,
- **stateless** so any server can handle any request, and
- **extensible** without breaking old clients.

HTTP has served this role since the early 1990s. Its design choices (text-based headers,
uniform methods, status codes, caching rules) all follow from those needs.

---

## 2. The mental model: request and response messages

HTTP is a **request/response** protocol. The client sends a request; the server sends
exactly one response. Here's a real exchange, as it looks on the wire in HTTP/1.1:

```http
POST /api/tickets HTTP/1.1
Host: beacon.example.com
Content-Type: application/json
Accept: application/json
Authorization: Bearer eyJhbGciOi...
Content-Length: 58

{"title":"Cannot log in","priority":"High","description":""}
```

```http
HTTP/1.1 201 Created
Content-Type: application/json; charset=utf-8
Location: /api/tickets/1042
Date: Wed, 07 Oct 2026 15:04:05 GMT
Content-Length: 97

{"id":"T-1042","title":"Cannot log in","status":"Open","priority":"High","assignee":null}
```

Every message has:

1. A **start line**: for requests, *method*, *target*, *version*; for responses, *version*,
   *status code*, *reason phrase*.
2. **Headers**: `Name: value` metadata, case-insensitive names.
3. A blank line.
4. An optional **body**, whose format is described by `Content-Type` and whose length is
   given by `Content-Length` or chunked encoding.

### Stateless

Each request is independent: the server doesn't remember the previous request from the
same client. Anything needed to process a request must be in it (or referenced by it: a
cookie or token pointing to stored state). Statelessness is why HTTP services scale
horizontally: any instance behind a load balancer can serve any request.

### URLs

```text
https://beacon.example.com:443/api/tickets?status=open&page=2#top
└─┬─┘   └───────┬────────┘└┬┘└────┬─────┘└────────┬────────┘└┬┘
scheme        host       port   path            query     fragment
```

The fragment (`#top`) never reaches the server; the browser keeps it. Query strings and
paths must be **percent-encoded** (`space` → `%20`). Never build URLs by string
concatenation with user input; use `Uri.EscapeDataString` or a builder.

---

## 3. Methods and their guarantees

HTTP methods carry *semantics* that clients, servers and intermediaries rely on. Two
properties matter most:

- **Safe**: the request doesn't change server state (from the client's point of view).
  Crawlers, prefetchers and caches may issue safe requests freely.
- **Idempotent**: sending the request N times has the same effect as sending it once.
  Clients and proxies may **automatically retry** idempotent requests after a network
  failure.

| Method | Purpose | Safe | Idempotent | Body |
|---|---|---|---|---|
| `GET` | Read a resource | ✓ | ✓ | No |
| `HEAD` | Like GET, headers only | ✓ | ✓ | No |
| `OPTIONS` | Describe capabilities (CORS preflight) | ✓ | ✓ | Rarely |
| `POST` | Create, or perform a non-idempotent action | ✗ | ✗ | Yes |
| `PUT` | Replace a resource entirely | ✗ | ✓ | Yes |
| `PATCH` | Partially modify a resource | ✗ | ✗ (can be made so) | Yes |
| `DELETE` | Remove a resource | ✗ | ✓ | Rarely |

Why it matters in practice:

- **Never change state on `GET`.** A link like `GET /tickets/42/close` will eventually be
  followed by a crawler, a link preview in a chat app, or a browser prefetch.
- **`PUT` is idempotent; `POST` isn't.** Sending "set ticket 42's title to X" twice is
  harmless. Sending "create a ticket" twice creates two. If a `POST` times out, the client
  doesn't know whether it succeeded. Chapter 4 covers **idempotency keys**, the standard
  solution.
- **`DELETE` is idempotent in effect** even if the second call returns `404`: the
  resource is gone either way.

---

## 4. Status codes

The status code tells the client *what kind* of outcome happened, in a way any client
understands without reading the body.

| Range | Meaning | Who's responsible |
|---|---|---|
| **1xx** | Informational | |
| **2xx** | Success | |
| **3xx** | Redirection: look elsewhere | |
| **4xx** | Client error: the request is wrong | The client: retrying the same request won't help |
| **5xx** | Server error: the server failed | The server: retrying later might help |

The ones you'll use constantly:

| Code | Meaning | Use when |
|---|---|---|
| 200 OK | Success with a body | GET, or an update returning the resource |
| 201 Created | Resource created | POST that creates; include `Location` |
| 202 Accepted | Accepted for async processing | Long-running work queued |
| 204 No Content | Success, no body | DELETE, or PUT returning nothing |
| 301 / 308 | Moved permanently (308 keeps the method) | Changed URLs |
| 302 / 307 | Temporary redirect (307 keeps the method) | Login redirects |
| 304 Not Modified | Cached version is still valid | Conditional GET (section 6) |
| 400 Bad Request | Malformed or invalid input | Validation failures |
| 401 Unauthorized | **Not authenticated**: who are you? | Missing or invalid credentials |
| 403 Forbidden | **Not authorized**: you can't do this | Authenticated but lacking permission |
| 404 Not Found | No such resource | Also used to hide existence from unauthorized users |
| 405 Method Not Allowed | Method not supported here | `DELETE` on a read-only resource |
| 409 Conflict | Conflicts with current state | Closed ticket, duplicate, version conflict |
| 412 Precondition Failed | `If-Match` failed | Optimistic concurrency (Chapter 4) |
| 415 Unsupported Media Type | Wrong `Content-Type` | XML sent to a JSON API |
| 422 Unprocessable Content | Well-formed but semantically invalid | Some APIs use this for validation |
| 429 Too Many Requests | Rate limited | Include `Retry-After` |
| 500 Internal Server Error | Unhandled server failure | Bugs |
| 502 Bad Gateway | A proxy got a bad response upstream | Your app crashed behind Nginx |
| 503 Service Unavailable | Temporarily unable | Overload, maintenance; include `Retry-After` |
| 504 Gateway Timeout | A proxy timed out waiting upstream | Your app is too slow behind a proxy |

> **⚠️ What can go wrong:** Returning `200 OK` with `{"success": false, "error": "..."}`
> breaks every generic tool: monitoring counts it as success, retry policies don't retry,
> caches may store it. Use status codes for what they mean, and put details in the body.

---

## 5. Headers that matter

Headers carry metadata for clients, servers and every intermediary.

### Content negotiation

| Header | Direction | Purpose |
|---|---|---|
| `Content-Type` | both | Format of *this* body: `application/json; charset=utf-8` |
| `Accept` | request | Formats the client can handle |
| `Content-Encoding` | response | Compression: `gzip`, `br` |
| `Accept-Encoding` | request | Compressions the client supports |
| `Accept-Language` | request | Preferred languages |

### Identity and context

| Header | Purpose |
|---|---|
| `Authorization` | Credentials: `Bearer <token>`, `Basic ...` |
| `Cookie` / `Set-Cookie` | Browser state (sessions); see Chapter 6 |
| `User-Agent` | Client software |
| `Host` | Which site (one IP can host many) |
| `X-Forwarded-For`, `X-Forwarded-Proto` / `Forwarded` | Original client IP and scheme when behind a proxy (Book VIII) |
| `traceparent` | W3C trace context for distributed tracing (Book IX) |

### Caching and conditional requests

`Cache-Control`, `ETag`, `If-None-Match`, `Last-Modified`, `If-Modified-Since`, `Vary`
(next section).

### Security

`Strict-Transport-Security`, `Content-Security-Policy`, `X-Content-Type-Options`,
CORS headers (`Access-Control-Allow-Origin`, ...). Chapter 9 covers them.

---

## 6. HTTP caching

Caching is one of HTTP's most powerful features and most misunderstood. Responses can be
cached by the **browser** (private cache), and by **shared caches** (CDNs, reverse
proxies).

### `Cache-Control`

```http
Cache-Control: public, max-age=3600          # anyone may cache for an hour
Cache-Control: private, max-age=60           # only the browser, for a minute
Cache-Control: no-cache                      # may store, but must revalidate before each use
Cache-Control: no-store                      # never store (sensitive data)
```

Note that `no-cache` does **not** mean "don't cache"; that's `no-store`.

### Validation with ETags

An **ETag** is an identifier for a specific version of a resource. The server sends it;
the client sends it back on the next request:

```http
GET /api/tickets/1042                        HTTP/1.1 200 OK
                                             ETag: "v7"
                                             { ...ticket... }

GET /api/tickets/1042                        HTTP/1.1 304 Not Modified
If-None-Match: "v7"                          (no body: use your cached copy)
```

The server still processes the request, but if nothing changed it sends no body, saving
bandwidth. The same mechanism powers **optimistic concurrency** for updates
(`If-Match: "v7"` → `412 Precondition Failed` if someone else changed it); Chapter 4 uses it.

### `Vary`

If a response depends on a request header (for example `Accept-Language` or
`Authorization`), the server must say so with `Vary`, or a shared cache may serve one
user's response to another.

> **⚠️ What can go wrong:** A CDN caching a personalized response, such as `/api/me`
> without `Cache-Control: private`, can serve one user's data to every other user. For
> authenticated API responses, default to `Cache-Control: no-store` or `private`, and
> cache publicly only what is truly public.

---

## 7. Connections and protocol versions

HTTP's *semantics* (methods, status codes, headers) have stayed stable. Its *transport*
has evolved substantially:

| Version | Transport | Key characteristics |
|---|---|---|
| **HTTP/1.1** (1997) | TCP | Text format; persistent connections (keep-alive); one request at a time per connection, so browsers open ~6 connections per host |
| **HTTP/2** (2015) | TCP + TLS | Binary framing; **multiplexing** many requests over one connection; header compression (HPACK); used by gRPC |
| **HTTP/3** (2022) | **QUIC over UDP** | Multiplexing without TCP head-of-line blocking; faster connection setup; survives network changes (Wi-Fi → mobile) |

What happens before the first byte of HTTP:

```text
 DNS lookup ─► TCP handshake (1 RTT) ─► TLS handshake (1 RTT in TLS 1.3) ─► HTTP request
 (name → IP)                                                               ─► response
```

On a 100 ms round trip, a fresh HTTPS connection costs ~200 ms before the request is even
sent. That's why **connection reuse** matters so much, and why creating a new
`HttpClient` per request is a performance problem (Chapter 2 and section 9).

> **🔄 Current (as of October 2026):** Kestrel (ASP.NET Core's server) supports HTTP/1.1,
> HTTP/2 and HTTP/3. HTTP/3 requires QUIC support from the OS (MsQuic, included on
> Windows and available as `libmsquic` on Linux). In most deployments, a reverse proxy or
> load balancer terminates HTTP/3 and HTTP/2 and talks to the app over HTTP/1.1 or HTTP/2.

### HTTPS

HTTP over **TLS** encrypts and authenticates the connection: no one in the middle can
read or modify traffic, and the client verifies it's talking to the real server via a
certificate. Every production API must use HTTPS only; Book VIII, Chapter 3 covers TLS
and certificates in depth.

---

## 8. Beyond request/response

Some interactions don't fit "client asks, server answers":

- **Polling**: the client asks repeatedly. Simple, wasteful.
- **Long polling**: the server holds the request open until there's news.
- **Server-Sent Events (SSE)**: the server streams events over one long-lived HTTP
  response (`Content-Type: text/event-stream`). One-directional, simple, works through
  proxies. Widely used for streaming AI responses (Book XI).
- **WebSockets**: an HTTP request *upgrades* the connection to a full-duplex,
  message-based channel. Chats, collaborative editing, live dashboards.

Chapter 8 implements real-time features for Beacon.

---

## 9. In practice: seeing HTTP for real

Before building Beacon's API (next chapter), get comfortable inspecting raw HTTP.

**curl** shows full requests and responses with `-v`:

```bash
curl -v https://api.github.com/repos/dotnet/runtime \
  -H "Accept: application/vnd.github+json"
```

Read the output: the TLS handshake (`* SSL connection using TLSv1.3`), the request lines
(`> GET ...`), the response status and headers (`< HTTP/2 200`, `< etag: ...`,
`< cache-control: ...`). Then try a conditional request with the ETag you received:

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://api.github.com/repos/dotnet/runtime \
  -H 'If-None-Match: "<etag-from-previous-response>"'
# 304
```

**Browser DevTools** (Network tab) show every request a page makes, with timing
(DNS, connect, TLS, waiting for the server, download), headers, and whether responses came
from cache.

**`.http` files** in Visual Studio, VS Code (REST Client extension) and Rider let you keep
runnable requests next to your code. Beacon's API will have one:

```http
### beacon.http
@baseUrl = https://localhost:7001

GET {{baseUrl}}/api/tickets/1042
Accept: application/json

###
POST {{baseUrl}}/api/tickets
Content-Type: application/json

{ "title": "Cannot log in", "priority": "High" }
```

### `HttpClient` done right

When Beacon calls other HTTP services (sending notifications, calling AI APIs in Book XI),
it will use `HttpClient`. The essentials:

```csharp
// ✗ A new client per call: new connections each time, risks socket exhaustion
using var client = new HttpClient();

// ✓ Typed client via IHttpClientFactory: pooled connections, configurable, testable
builder.Services.AddHttpClient<SlackNotifier>(c =>
{
    c.BaseAddress = new Uri("https://hooks.slack.com/");
    c.Timeout = TimeSpan.FromSeconds(10);
})
.AddStandardResilienceHandler();   // retries with back-off, circuit breaker, timeouts
```

`AddStandardResilienceHandler` (from `Microsoft.Extensions.Http.Resilience`) retries only
**idempotent-safe** failures by default (transient errors, 5xx, 408, 429), with
exponential back-off and jitter. That's section 3's idempotency rules, applied
automatically.

---

## 10. What can go wrong

- **State changes on `GET`**, triggered by crawlers and prefetchers.
- **Retrying non-idempotent requests**, creating duplicates.
- **Wrong status codes**: 200 for errors, 500 for validation, 401 vs 403 confusion.
- **Caching personalized data publicly**, or missing `Vary`.
- **New connection per request** (`new HttpClient()`), causing latency and port exhaustion.
- **Ignoring timeouts.** The default `HttpClient.Timeout` is 100 seconds; a hung
  dependency can hold your threads and requests for that long.
- **Mismatched proxies and apps**: the app thinks requests are HTTP (not HTTPS) or sees the
  proxy's IP as the client's, because forwarded headers aren't processed (Book VIII).

---

## 11. How an experienced engineer thinks about this

- **Respect method semantics.** Safety and idempotency aren't pedantry; infrastructure
  relies on them.
- **Status codes are an API.** Clients, proxies, monitoring and retry logic all read them.
- **Assume intermediaries exist.** Caches, CDNs and proxies will act on your headers;
  set them deliberately.
- **Latency is mostly connections and round trips.** Reuse connections; reduce chattiness.
- **When in doubt, look at the wire.** `curl -v` and DevTools settle most arguments.

---

## 12. Check yourself

**Questions**

1. What makes a method safe? Idempotent? Classify GET, POST, PUT, PATCH and DELETE.
2. Why shouldn't a `GET` request change state?
3. What's the difference between 401 and 403? Between 400 and 409?
4. What's the difference between `no-cache` and `no-store`?
5. How do ETags enable both caching and optimistic concurrency?
6. What does HTTP/2 multiplexing solve? What does HTTP/3 add?
7. Why is creating a new `HttpClient` per request a problem?

**Exercises**

1. Use `curl -v` against a public API and identify every header in the response. Look up
   any you don't recognize.
2. In browser DevTools, load a site twice and find which resources came from cache and
   which were revalidated with `304`.
3. Write a `.http` file with five requests for a public API (GET with query, conditional
   GET, a 404, etc.).
4. Explain, step by step, what happens when a mobile client's `POST /orders` times out and
   it retries.

**Interview-style questions**

- "What happens when you type a URL into a browser and press Enter?"
- "What's the difference between PUT and PATCH? Between PUT and POST?"
- "How does HTTP caching work?"
- "What's idempotency, and why does it matter for APIs?"

---

## 13. Going deeper

- [MDN: HTTP](https://developer.mozilla.org/docs/Web/HTTP) — the best general reference.
- RFC 9110 (HTTP Semantics) and RFC 9111 (HTTP Caching) — the actual specifications,
  surprisingly readable.
- *High Performance Browser Networking* by Ilya Grigorik (free online) — connections,
  TLS, HTTP/2 and performance.

**Next:** [Chapter 2 — ASP.NET Core Fundamentals](02-asp-net-core-fundamentals.md) builds
Beacon's API on top of this foundation.
