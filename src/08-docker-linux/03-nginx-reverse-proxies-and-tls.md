# Nginx, Reverse Proxies and TLS

Beacon.Api listens on `127.0.0.1:5000`: safe, and unreachable. To serve users, something
must accept connections from the internet on ports 80 and 443, terminate TLS, and pass
requests to the app. That something is a **reverse proxy**, and on Linux servers it's
very often **Nginx**.

The reverse proxy sits on the most important seam in the system: every request passes
through it. Misconfigured, it causes some of the most confusing bugs in web development:
redirect loops, wrong client IPs in logs, "insecure" cookies, broken WebSockets, 502s and
504s. This chapter explains what a reverse proxy does, how TLS works, and how to configure
both correctly for an ASP.NET Core application.

---

## 1. The problem: the edge of the system

Whatever faces the internet needs to:

- terminate **TLS** with valid certificates, renewed automatically,
- redirect HTTP to HTTPS,
- route requests to the right backend (API, BFF, static files),
- serve static files efficiently, with compression and caching headers,
- protect backends: request size limits, timeouts, rate limits, buffering slow clients,
- support WebSockets for real-time features,
- balance load across multiple app instances,
- log every request.

Kestrel can do some of this, but a dedicated proxy does it better, and lets several apps
share one public IP and certificate.

---

## 2. The mental model: forward vs reverse proxies

```text
 Forward proxy (acts for clients):    clients ──► proxy ──► any server on the internet
 Reverse proxy (acts for servers):    any client ──► proxy ──► your backend servers
```

A **reverse proxy** receives requests on behalf of your servers. Clients don't know (or
care) what's behind it.

```text
                         ┌────────────────────── server ──────────────────────┐
 Browser ──HTTPS:443───► │ Nginx                                               │
                         │  ├─ TLS termination                                  │
                         │  ├─ /            → static SPA files (or the BFF)     │
                         │  ├─ /api, /hubs  → http://127.0.0.1:5001 (BFF)       │
                         │  └─ logs, limits, compression                        │
                         │                       Beacon.Bff ──► Beacon.Api      │
                         └─────────────────────────────────────────────────────┘
```

The same role is played by cloud load balancers (Azure Application Gateway, Front Door,
AWS ALB), Kubernetes ingress controllers, and Caddy, Traefik, HAProxy or YARP. The concepts
in this chapter transfer to all of them.

---

## 3. TLS: how HTTPS works

**TLS** (Transport Layer Security) provides three guarantees:

1. **Confidentiality**: traffic is encrypted.
2. **Integrity**: traffic can't be modified undetected.
3. **Authentication**: the client knows it's talking to the real `beacon.example.com`, not an
   impostor.

### Certificates and trust

A **certificate** binds a public key to a domain name, signed by a **Certificate Authority
(CA)**. Browsers and operating systems ship a list of trusted root CAs. The chain:

```text
 Root CA (in the browser's trust store)
   └─ signs Intermediate CA certificate
        └─ signs beacon.example.com certificate (leaf)  ← your server presents leaf + intermediate
```

The client verifies the chain up to a trusted root, checks that the name matches the
hostname it requested, and that the certificate hasn't expired.

### The handshake (TLS 1.3), simplified

```text
 Client                                         Server
 ClientHello (supported ciphers, key share) ──►
                                            ◄── ServerHello (chosen cipher, key share),
                                                certificate, signature proving key ownership
 [both derive the same session keys from the key shares]
 Finished ─────────────────────────────────────►
 ◄═══════════════ encrypted HTTP traffic ═══════════════►
```

TLS 1.3 completes in one round trip. Only the certificate's **public** key is sent; the
**private key** stays on the server and must be protected (file permissions `600`, or a key
vault).

### Let's Encrypt and automation

**Let's Encrypt** issues free, domain-validated certificates via the **ACME** protocol.
Certificates are short-lived (90 days, and the industry is moving toward even shorter
lifetimes), which makes **automatic renewal mandatory**. Tools: **Certbot**, Caddy
(automatic HTTPS built in), cert-manager in Kubernetes, and managed certificates in cloud
load balancers.

> **⚠️ What can go wrong:** Expired certificates are one of the most common causes of total
> outages, including at large companies. Automate renewal, and monitor certificate expiry
> independently (alert at 14 days remaining), because renewal automation fails silently too.

### TLS termination

Where TLS ends:

| Pattern | TLS from client ends at | Proxy → app traffic |
|---|---|---|
| **Termination at the proxy** | The proxy | Plain HTTP on a trusted network (localhost or private network) |
| **Re-encryption** | The proxy, then a new TLS connection | HTTPS to the app (required by some compliance standards, zero-trust networks) |
| **Passthrough** | The app | The proxy only forwards TCP |

Beacon terminates at Nginx and talks plain HTTP to apps on `127.0.0.1`. Across a network
(proxy and app on different machines), re-encrypt or use a private network you trust.

---

## 4. Configuring Nginx

### Structure

```text
/etc/nginx/nginx.conf                 global settings (worker processes, logging, includes)
/etc/nginx/sites-available/beacon     your site's config
/etc/nginx/sites-enabled/beacon       symlink to enable it
```

```bash
sudo nginx -t                         # test configuration: always before reloading
sudo systemctl reload nginx           # apply without dropping connections (SIGHUP)
```

### Beacon's site configuration

```nginx
# /etc/nginx/sites-available/beacon

# Redirect all HTTP to HTTPS (and serve ACME challenges for certificate renewal)
server {
    listen 80;
    listen [::]:80;
    server_name beacon.example.com;

    location /.well-known/acme-challenge/ { root /var/www/certbot; }
    location / { return 301 https://$host$request_uri; }
}

upstream beacon_bff {
    server 127.0.0.1:5001;
    keepalive 32;                                   # reuse connections to the app
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name beacon.example.com;

    ssl_certificate     /etc/letsencrypt/live/beacon.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/beacon.example.com/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_session_cache shared:SSL:10m;

    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    server_tokens off;                              # don't advertise the Nginx version

    client_max_body_size 2m;                        # Book III, Chapter 9: limit request bodies
    gzip on;
    gzip_types application/json text/css application/javascript image/svg+xml;

    # Hashed static assets from the SPA build: cache forever
    location /assets/ {
        root /opt/beacon/web;
        add_header Cache-Control "public, max-age=31536000, immutable";
        try_files $uri =404;                        # a missing chunk is a 404, not index.html
    }

    # Real-time (SignalR WebSockets) through the BFF
    location /hubs/ {
        proxy_pass http://beacon_bff;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 1h;                      # long-lived connections
    }

    # Everything else (API, BFF endpoints, SPA routes) to the BFF
    location / {
        proxy_pass http://beacon_bff;
        proxy_http_version 1.1;
        proxy_set_header Connection "";             # enable upstream keepalive
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host $host;
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
    }
}
```

Key points:

- **`try_files $uri =404` for `/assets/`**: after a deployment, an old tab requesting a
  deleted chunk gets a real 404 (which the SPA can handle by reloading), not `index.html`
  served with status 200, which would fail with a confusing "unexpected token <" error.
- **WebSocket upgrade headers** are required for SignalR; without them it silently falls
  back to slower transports or fails.
- **`proxy_read_timeout`** must exceed your slowest legitimate request, and the long-lived
  hub connections.
- **HSTS** tells browsers to always use HTTPS for this domain.

### Getting certificates

```bash
sudo apt install -y certbot
sudo certbot certonly --webroot -w /var/www/certbot -d beacon.example.com
systemctl list-timers | grep certbot          # renewal timer installed by the package
sudo certbot renew --dry-run
```

Add a deploy hook to reload Nginx after renewal (`--deploy-hook "systemctl reload nginx"`).

---

## 5. Forwarded headers: making the app aware of the proxy

Behind a proxy, the app sees every request coming from `127.0.0.1` over plain HTTP. Without
correction:

- **Logs and rate limiting** see the proxy's IP, not the client's: every user looks like
  one user.
- **`Request.Scheme` is `http`**, so generated URLs (redirects, `Location` headers, OIDC
  callback URLs) use `http://`, causing redirect loops or OIDC failures.
- **Secure cookies** may not be issued, or the app thinks the connection is insecure.

The proxy passes the original values in **forwarded headers**; the app must be told to
**trust them, but only from the proxy**:

```csharp
// Beacon.Bff and Beacon.Api Program.cs
using Microsoft.AspNetCore.HttpOverrides;

builder.Services.Configure<ForwardedHeadersOptions>(o =>
{
    o.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto | ForwardedHeaders.XForwardedHost;
    o.KnownProxies.Clear();
    o.KnownProxies.Add(IPAddress.Loopback);           // only trust headers from the local Nginx
    o.AllowedHosts = ["beacon.example.com"];
});

var app = builder.Build();
app.UseForwardedHeaders();                            // first in the pipeline
```

> **⚠️ What can go wrong:** Trusting `X-Forwarded-For` from *anyone* lets clients spoof their
> IP address, bypassing IP-based rate limits and polluting audit logs. Only trust forwarded
> headers from known proxy addresses (`KnownProxies`/`KnownNetworks`). And `X-Forwarded-For`
> is a list: the client IP is the right-most entry added by a *trusted* proxy, not
> necessarily the first.

---

## 6. Load balancing and health

With several app instances, Nginx distributes requests:

```nginx
upstream beacon_api {
    least_conn;                                       # send to the instance with fewest active connections
    server 10.0.1.11:5000 max_fails=3 fail_timeout=10s;
    server 10.0.1.12:5000 max_fails=3 fail_timeout=10s;
    keepalive 64;
}
```

Load balancing algorithms: **round robin** (default), **least connections**, **IP hash**
(crude stickiness). Open-source Nginx marks backends as failed passively (after errors);
active health checks (probing `/health/ready`, Book III, Chapter 10) are a feature of
cloud load balancers, Kubernetes, and commercial or alternative proxies.

Statelessness pays off here (Book III, Chapter 1): any instance can serve any request, so
the balancer can use any algorithm. Things that break statelessness, like in-memory sessions
or in-memory caches without invalidation, or SignalR without a backplane (Book III,
Chapter 8), need special handling.

### Zero-downtime deploys behind a proxy

1. Take one instance out of rotation (or let the health check fail via a "draining" state).
2. Wait for in-flight requests to finish (graceful shutdown, Chapter 2).
3. Deploy and start the new version; wait for `/health/ready`.
4. Put it back; repeat for the next instance.

Container platforms automate exactly this (Book IX).

---

## 7. Diagnosing proxy errors

> **🔍 Investigation: 502 and 504 errors.**
> - **502 Bad Gateway**: Nginx couldn't get a valid response. Check the Nginx error log
>   (`/var/log/nginx/error.log`): "connect() failed (111: Connection refused)" means the app
>   isn't listening (crashed, restarting, wrong port); "upstream prematurely closed
>   connection" means the app died mid-request. Then `systemctl status` and `journalctl` for
>   the app.
> - **504 Gateway Timeout**: the app accepted the connection but didn't respond within
>   `proxy_read_timeout`. The app is slow or stuck: Book III, Chapter 10's playbook (slow
>   queries, thread-pool starvation, dependency timeouts).
> - **413 Request Entity Too Large**: `client_max_body_size` (deliberately) rejected an
>   upload. Raise it for the specific upload location only.
> - **Redirect loops / OIDC callback errors**: forwarded headers not processed, so the app
>   thinks it's on `http`.

The access log plus the app's trace ID connect the two sides. Add the request ID to Nginx's
log format and forward it (`proxy_set_header X-Request-Id $request_id;`) so one ID appears
in both logs.

---

## 8. In practice: Beacon on the internet

Putting Chapters 1–3 together on one server:

1. **Build** the SPA (`pnpm --dir web build`) and publish Beacon.Bff and Beacon.Api.
2. **Deploy** to `/opt/beacon/{bff,api,web}`, owned by the `beacon` user.
3. **Run** `beacon-api.service` (`127.0.0.1:5000`) and `beacon-bff.service` (`127.0.0.1:5001`,
   with the BFF's YARP proxy pointing at the API).
4. **Point DNS**: an A record for `beacon.example.com` to the server's public IP.
5. **Obtain a certificate** with Certbot and enable the Nginx site.
6. **Configure forwarded headers** in both apps.
7. **Firewall**: only 22 (from your IP), 80 and 443 open.

Verify from outside:

```bash
curl -sSI http://beacon.example.com          # 301 → https
curl -sSI https://beacon.example.com         # 200, HSTS header, no "Server: nginx/1.x"
curl -sS https://beacon.example.com/health/ready
openssl s_client -connect beacon.example.com:443 -servername beacon.example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -dates -issuer        # certificate dates and issuer
```

Check the app sees real client IPs and HTTPS (a log line at startup of the first request,
or a temporary diagnostic endpoint in a non-production environment).

This is a complete, production-capable single-server deployment, the kind many real
products run on for years. Chapters 4–6 package it into containers, and Book IX moves it to
managed cloud services.

---

## 9. What can go wrong

- **Expired certificates** without monitoring.
- **Missing forwarded-headers processing**: wrong IPs, redirect loops, insecure cookies.
- **Trusting forwarded headers from anyone.**
- **WebSocket upgrades not configured.**
- **SPA fallback serving `index.html` for missing assets.**
- **Timeouts mismatched** between proxy, app and clients.
- **Private keys readable by other users.**
- **Reloading with an invalid config** (always `nginx -t` first).

---

## 10. How an experienced engineer thinks about this

- **The edge is a security boundary**: TLS, headers, limits, minimal exposure.
- **Automate certificates and monitor them anyway.**
- **Make the app proxy-aware**, trusting only known proxies.
- **Keep apps stateless** so any instance can serve any request.
- **Read both logs**: proxy and app, linked by a request ID.

---

## 11. Check yourself

**Questions**

1. What's the difference between a forward and a reverse proxy?
2. What three guarantees does TLS provide? What does a certificate prove?
3. Why must certificate renewal be automated, and monitored?
4. What goes wrong if an app behind a proxy doesn't process forwarded headers?
5. Why only trust forwarded headers from known proxies?
6. What Nginx settings does SignalR need?
7. What's the difference between a 502 and a 504 from Nginx?

**Exercises**

1. Configure Nginx in front of Beacon on a VM with a Let's Encrypt certificate (or a
   self-signed certificate locally) and test the redirect, HSTS and WebSockets.
2. Remove `UseForwardedHeaders` temporarily and observe the effect on the OIDC login flow
   and logged client IPs.
3. Add `$request_id` to the Nginx log format and the app's logs, and follow one request
   through both.
4. Run two instances of Beacon.Api behind an Nginx upstream and stop one during a load test.

**Interview-style questions**

- "Explain how HTTPS works."
- "What does a reverse proxy do, and why would you put one in front of your app?"
- "Your users see 502 errors. How do you investigate?"

---

## 12. Going deeper

- [Nginx documentation](https://nginx.org/en/docs/) and the
  [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
- [Microsoft docs: Configure ASP.NET Core to work with proxy servers and load balancers](https://learn.microsoft.com/aspnet/core/host-and-deploy/proxy-load-balancer)
- [Let's Encrypt documentation](https://letsencrypt.org/docs/)
- *Bulletproof TLS and PKI* by Ivan Ristić.

**Next:** [Chapter 4 — Containers and Images](04-containers-and-images.md) packages
Beacon so it runs the same way everywhere.
