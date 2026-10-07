# Concurrency, Async and Networking

Rust's marketing phrase is **"fearless concurrency"**: the same ownership rules that prevent
memory bugs (Chapter 2) also prevent **data races at compile time**. Code that shares mutable
state between threads without synchronization doesn't compile. Combined with an efficient async
model, that makes Rust well suited to highly concurrent network services.

This chapter covers threads and data parallelism (making `beacon-logscan` use every core), async
Rust with Tokio, and builds `beacon-relay`, a webhook delivery service, comparing each concept with
.NET (Book I, Chapters 9–10).

> **🔄 Current (as of October 2026):** Code in this chapter was compiled and tested with Rust 1.97
> (2024 edition), Tokio 1.x, axum 0.8 and reqwest 0.13. Crate APIs change between major versions;
> check their documentation if you use newer versions.

---

## 1. The problem: concurrency bugs are hard to find

Book I, Chapter 10 listed the hazards: race conditions on shared state, deadlocks, visibility problems,
and the difficulty of testing timing-dependent bugs. In C#, avoiding data races is a matter of
discipline and review. Rust moves the most dangerous class, **data races**, into the type system.

---

## 2. The mental model: `Send`, `Sync` and ownership across threads

Two marker traits, implemented automatically by the compiler:

- **`Send`**: a value of this type can be **moved** to another thread.
- **`Sync`**: a value of this type can be **referenced** from several threads at once (`&T` is `Send`).

Most types are both. Exceptions are exactly the dangerous ones:

| Type | Send | Sync | Why |
|---|---|---|---|
| `Rc<T>` | ✗ | ✗ | Non-atomic reference count would race |
| `RefCell<T>` | ✓ | ✗ | Run-time borrow tracking isn't thread-safe |
| `Arc<T>` | ✓ (if T: Send + Sync) | ✓ | Atomic reference count |
| `Mutex<T>` | ✓ | ✓ | Locking makes shared mutation safe |
| Raw pointers | ✗ | ✗ | No guarantees |

`std::thread::spawn` requires its closure (and everything it captures) to be `Send + 'static`. Try to
share a `Vec` mutably between two threads without a `Mutex`, and the program doesn't compile:

```rust
let mut counts = vec![0u64; 4];
std::thread::scope(|s| {
    s.spawn(|| counts[0] += 1);
    s.spawn(|| counts[1] += 1);   // ✗ error: cannot borrow `counts` as mutable more than once
});
```

The same rule from Chapter 2 (one mutable borrow at a time) applies across threads. Fixes are the usual
ones, now enforced: give each thread its own data (split the slice with `split_at_mut` or `chunks_mut`),
use atomics, or wrap shared state in `Mutex`/`RwLock`.

> **🧱 Durable:** Rust prevents **data races** (unsynchronized concurrent access to memory), not all
> **race conditions** (logic that depends on timing, like check-then-act across two lock acquisitions)
> and not deadlocks. Book I, Chapter 10's design advice still applies: avoid sharing, prefer message
> passing, lock briefly and in a consistent order.

---

## 3. Data parallelism with Rayon

For CPU-bound work over collections, **Rayon** provides parallel iterators: change `iter()` to
`par_iter()`, and work is spread across a thread pool with work stealing, like `Parallel.ForEach` and
PLINQ (Book I, Chapter 10):

```rust
use rayon::prelude::*;

// One file per task, on all cores. Each task owns its own report: nothing shared, nothing locked.
let reports: Vec<_> = args
    .files
    .par_iter()
    .map(|path| scan::scan_file::<HistogramSummary>(path, args.max_bad_pct).map(|r| (path, r)))
    .collect::<Result<_, _>>()?;          // the first error (if any) is returned

let mut total = scan::FileReport::<HistogramSummary>::default();
for (path, r) in &reports {
    println!("{}: {} lines, {} 5xx, p95 ≈ {:.0} ms",
             path.display(), r.total, r.server_errors, r.latency.p95().unwrap_or(0.0));
    total.merge(r);
}
```

```rust
// merging two histograms: add bucket counts
impl HistogramSummary {
    pub fn merge(&mut self, other: &Self) {
        for (a, b) in self.buckets.iter_mut().zip(other.buckets.iter()) {
            *a += b;
        }
        self.count += other.count;
    }
}
```

The design is the "don't share" option from Book I, Chapter 10: each file is processed independently into
its own report, and reports are **merged** at the end. That's why the histogram summary (Chapter 3) was a
good choice: histograms merge exactly, while exact percentiles would need all samples. Rayon checks the
closure is `Send + Sync` at compile time, so accidentally sharing a mutable counter is impossible.

On a machine with 8 cores, scanning 8 large daily log files takes roughly the time of one.

---

## 4. Async Rust

### Why async

Book I, Chapter 9's argument applies unchanged: I/O-bound services spend most of their time waiting, and
OS threads are too expensive to dedicate one per waiting operation. Async lets a few threads multiplex
thousands of in-flight operations.

### The model: futures are lazy state machines

```rust
async fn fetch_status(client: &reqwest::Client, url: &str) -> reqwest::Result<u16> {
    let response = client.get(url).send().await?;
    Ok(response.status().as_u16())
}
```

Like C#, the compiler turns an `async fn` into a **state machine** (Book I, Chapter 9). Differences:

| | C# `Task` | Rust `Future` |
|---|---|---|
| Starts | When called (hot) | **When polled** (lazy): nothing happens until awaited or spawned |
| Runtime | Built into .NET (thread pool) | **Library you choose**: Tokio is the standard for servers |
| Allocation | A `Task` object per call (mostly) | State machine is a value; no allocation unless boxed or spawned |
| Cancellation | `CancellationToken` passed explicitly | **Dropping a future cancels it** at its next `.await` point |
| Spawning | `Task.Run` | `tokio::spawn(future)` (requires `Send + 'static`) |
| Blocking inside async | Thread-pool starvation | Same problem: blocks a runtime worker; use `spawn_blocking` |

Lazy futures and drop-to-cancel are the big conceptual differences. Calling an async function without
awaiting it does **nothing** (the compiler warns about unused futures). Cancellation is implicit and
pervasive: wrapping a future in `tokio::time::timeout(d, fut)` and letting it expire simply drops it.

### Tokio essentials

```rust
#[tokio::main]                                   // starts a multi-threaded runtime
async fn main() { /* ... */ }

tokio::spawn(async move { ... });                // concurrent task (like Task.Run for async work)
tokio::join!(a, b);                              // await several futures concurrently (like Task.WhenAll)
tokio::select! { r = a => ..., _ = shutdown => ... }   // first to complete wins; others are dropped
tokio::time::sleep(d).await;                     // async delay
tokio::time::timeout(d, fut).await;              // deadline
tokio::sync::{mpsc, Mutex, Semaphore, RwLock, oneshot, broadcast};   // async-aware primitives
tokio::task::spawn_blocking(|| cpu_or_blocking_work());              // offload blocking work
```

The async primitives mirror .NET: `mpsc` channels are `Channel<T>` (Book I, Chapter 10), `Semaphore` is
`SemaphoreSlim`, `tokio::sync::Mutex` is for locks held across `.await` (prefer `std::sync::Mutex` for
short, non-async critical sections).

---

## 5. Networking: HTTP clients and servers

- **reqwest**: the standard async HTTP client (connection pooling, TLS, JSON, timeouts). Create **one
  client** and reuse it (the `HttpClient` lesson from Book III, Chapter 1).
- **axum**: a web framework built on Tokio and the `tower` middleware ecosystem. Routes, extractors (path,
  query, JSON, state) and middleware feel familiar to an ASP.NET Core minimal API developer (Book III,
  Chapter 2).
- **tower** middleware: timeouts, rate limiting, tracing, compression, like ASP.NET Core middleware.
- **tracing**: structured, span-based logging and diagnostics, with OpenTelemetry export (Book IX,
  Chapter 6).
- **sqlx** (async SQL with compile-time checked queries) for PostgreSQL access.

---

## 6. In practice: `beacon-relay`

### The problem

Beacon adds **webhooks**: customers register an HTTPS endpoint and receive events (`ticket.created`,
`ticket.resolved`). Delivering them has awkward properties:

- Customer endpoints are slow, flaky or down; deliveries need **retries with back-off**.
- One slow customer must not delay everyone else: **per-endpoint concurrency limits**.
- Volume can spike (bulk imports): **bounded queues and back-pressure**.
- Receivers must be able to verify authenticity: **HMAC signatures**.
- Receivers may see duplicates (at-least-once; Book III, Chapter 8): a **delivery ID** for idempotency.
- Thousands of mostly idle, waiting connections: a good fit for a small async service.

In Beacon's design, Beacon.Api writes `WebhookRequested` events to the **outbox** (Book IV, Chapter 5);
`beacon-relay` claims them with `FOR UPDATE SKIP LOCKED`, delivers them, and records the result. For
clarity, the code below receives deliveries over a small internal HTTP endpoint instead of the database;
the delivery logic is the same.

### The code

```toml
# Cargo.toml (dependencies)
[dependencies]
anyhow = "1"
axum = "0.8"
hex = "0.4"
hmac = "0.13"
reqwest = { version = "0.13", default-features = false, features = ["json", "rustls"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
sha2 = "0.11"
tokio = { version = "1", features = ["full"] }
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["json"] }
```

```rust
// src/main.rs
use std::{collections::HashMap, sync::Arc, time::Duration};

use axum::{extract::State, http::StatusCode, routing::post, Json, Router};
use hmac::{Hmac, KeyInit, Mac};
use serde::Deserialize;
use sha2::Sha256;
use tokio::sync::{mpsc, Mutex, Semaphore};

/// A webhook to deliver (in production: read from Beacon's outbox table).
#[derive(Debug, Clone, Deserialize)]
struct Delivery {
    id: String,             // outbox message id: also the idempotency key for receivers
    endpoint: String,       // customer's HTTPS URL (validated when registered)
    secret: String,         // per-endpoint signing secret
    payload: serde_json::Value,
}

#[derive(Clone)]
struct AppState {
    queue: mpsc::Sender<Delivery>,
}

/// Limits concurrent requests per customer endpoint, so one slow endpoint can't use every slot.
#[derive(Default)]
struct EndpointLimits {
    per_endpoint: Mutex<HashMap<String, Arc<Semaphore>>>,
}

impl EndpointLimits {
    async fn for_endpoint(&self, endpoint: &str) -> Arc<Semaphore> {
        let mut map = self.per_endpoint.lock().await;
        map.entry(endpoint.to_owned())
            .or_insert_with(|| Arc::new(Semaphore::new(4)))
            .clone()
    }
}

fn sign(secret: &str, body: &[u8]) -> String {
    let mut mac = Hmac::<Sha256>::new_from_slice(secret.as_bytes()).expect("HMAC accepts any key length");
    mac.update(body);
    hex::encode(mac.finalize().into_bytes())
}

async fn deliver(client: &reqwest::Client, d: &Delivery) -> Result<(), String> {
    let body = serde_json::to_vec(&d.payload).map_err(|e| e.to_string())?;
    let signature = sign(&d.secret, &body);

    for attempt in 1..=5u32 {
        let result = client
            .post(&d.endpoint)
            .header("Content-Type", "application/json")
            .header("Beacon-Delivery-Id", &d.id)
            .header("Beacon-Signature", format!("sha256={signature}"))
            .body(body.clone())
            .send()
            .await;

        match result {
            Ok(r) if r.status().is_success() => return Ok(()),
            Ok(r) if r.status().is_client_error() && r.status() != reqwest::StatusCode::TOO_MANY_REQUESTS => {
                return Err(format!("permanent failure: HTTP {}", r.status())); // don't retry 4xx (except 429)
            }
            Ok(r) => tracing::warn!(id = %d.id, attempt, status = %r.status(), "retryable response"),
            Err(e) => tracing::warn!(id = %d.id, attempt, error = %e, "request failed"),
        }

        if attempt < 5 {
            // exponential back-off with jitter: 1s, 2s, 4s, 8s (+ up to 25%)
            let base = Duration::from_secs(1 << (attempt - 1));
            tokio::time::sleep(base + base.mul_f64(jitter_fraction() * 0.25)).await;
        }
    }
    Err("gave up after 5 attempts".into())
}

fn jitter_fraction() -> f64 {
    // tiny dependency-free jitter source; use the `rand` crate in real code
    let nanos = std::time::SystemTime::now()
        .duration_since(std::time::UNIX_EPOCH)
        .map(|d| d.subsec_nanos())
        .unwrap_or(0);
    f64::from(nanos % 1000) / 1000.0
}

async fn enqueue(State(state): State<AppState>, Json(d): Json<Delivery>) -> StatusCode {
    match state.queue.try_send(d) {
        Ok(()) => StatusCode::ACCEPTED,
        Err(_) => StatusCode::SERVICE_UNAVAILABLE, // queue full (back-pressure) or shutting down
    }
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    tracing_subscriber::fmt().json().init();

    let client = reqwest::Client::builder()
        .timeout(Duration::from_secs(10))
        .connect_timeout(Duration::from_secs(3))
        .pool_max_idle_per_host(8)
        .build()?;

    let (tx, mut rx) = mpsc::channel::<Delivery>(10_000); // bounded queue
    let limits = Arc::new(EndpointLimits::default());
    let global = Arc::new(Semaphore::new(200)); // at most 200 deliveries in flight

    // Dispatcher: one task per delivery, bounded globally and per endpoint.
    let dispatcher = tokio::spawn(async move {
        while let Some(d) = rx.recv().await {
            let permit = global.clone().acquire_owned().await.expect("semaphore never closed");
            let (client, limits) = (client.clone(), limits.clone());
            tokio::spawn(async move {
                let _global = permit; // released when this task ends
                let endpoint_limit = limits.for_endpoint(&d.endpoint).await;
                let _slot = endpoint_limit.acquire().await.expect("semaphore never closed");
                match deliver(&client, &d).await {
                    Ok(()) => tracing::info!(id = %d.id, "delivered"),
                    Err(e) => tracing::error!(id = %d.id, error = %e, "delivery failed"), // → mark failed in outbox
                }
            });
        }
    });

    let app = Router::new()
        .route("/deliveries", post(enqueue))
        .with_state(AppState { queue: tx });
    let listener = tokio::net::TcpListener::bind("127.0.0.1:8090").await?;
    tracing::info!("listening on 127.0.0.1:8090");
    axum::serve(listener, app)
        .with_graceful_shutdown(async {
            tokio::signal::ctrl_c().await.ok();
        })
        .await?;

    dispatcher.abort();
    Ok(())
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn signature_is_stable_hex_sha256() {
        let s = sign("secret", br#"{"ticket":"T-1"}"#);
        assert_eq!(s.len(), 64);
        assert_eq!(s, sign("secret", br#"{"ticket":"T-1"}"#));
        assert_ne!(s, sign("other", br#"{"ticket":"T-1"}"#));
    }
}
```

### Walking through it

- **Bounded channel** (`mpsc::channel(10_000)`): the HTTP endpoint uses `try_send` and returns `503` when
  full, applying **back-pressure** to the producer instead of growing memory without limit (Book I,
  Chapter 10).
- **Two levels of limits**: a global `Semaphore` caps in-flight deliveries at 200; a per-endpoint
  `Semaphore` caps each customer at 4 concurrent requests, so one slow endpoint can't monopolize capacity
  (the **bulkhead** pattern, Book IX, Chapter 7).
- **Ownership makes permits safe**: `acquire_owned()` returns a permit **owned** by the spawned task; when
  the task finishes (or panics, or is cancelled), the permit is dropped and released automatically. No
  `finally` blocks to forget.
- **Retry policy**: retry network errors, 5xx and 429 with exponential back-off and jitter; don't retry
  other 4xx (the receiver rejected the request; retrying won't help). Five attempts in this sketch; in
  production, the outbox row is rescheduled for much later (minutes to hours) rather than holding a task.
- **Signatures**: `Beacon-Signature: sha256=<HMAC of the body>` with a per-endpoint secret. Receivers
  recompute it to verify the request came from Beacon and wasn't modified. Include a timestamp in the
  signed content in production to prevent replay.
- **Timeouts** on connect (3 s) and the whole request (10 s): slow endpoints release resources quickly.
- **Structured JSON logs** via `tracing`, ready for Book IX's log pipeline.
- **The compiler checked** that everything moved into `tokio::spawn` is `Send + 'static`, and that no
  mutable state is shared without synchronization (the `HashMap` of semaphores sits behind a `Mutex`).

A smoke test against an unreachable endpoint shows the retry behavior:

```text
$ curl -X POST localhost:8090/deliveries -H 'content-type: application/json' \
    -d '{"id":"m1","endpoint":"http://127.0.0.1:9/hook","secret":"s","payload":{"ticket":"T-1"}}'
202
{"level":"WARN","fields":{"message":"request failed","id":"m1","attempt":1,...}}
{"level":"WARN","fields":{"message":"request failed","id":"m1","attempt":2,...}}   ← ~1 s later
{"level":"WARN","fields":{"message":"request failed","id":"m1","attempt":3,...}}   ← ~2 s later
```

### Security: SSRF

Webhook senders are a classic **SSRF** vector (Book III, Chapter 9): a customer registers
`http://169.254.169.254/...` or an internal hostname as their "endpoint." Defenses for `beacon-relay`:

- Accept only `https` URLs at registration; resolve the hostname and **reject private, loopback and
  link-local addresses**, both at registration and **at delivery time** (DNS can change).
- Don't follow redirects (or re-validate each hop).
- Run the relay in a network segment with **no access to internal services**, egressing through the NAT
  gateway (Book IX, Chapter 5).

### Deployment

The release binary is small and starts instantly. A Docker image on a minimal base (Book VIII, Chapter 6):

```dockerfile
FROM rust:1-slim AS build
WORKDIR /src
COPY . .
RUN cargo build --release --locked

FROM gcr.io/distroless/cc-debian12:nonroot
COPY --from=build /src/target/release/beacon-relay /beacon-relay
USER nonroot
ENTRYPOINT ["/beacon-relay"]
```

It runs as one more Container App in Beacon's environment (Book IX, Chapter 3), scaled by outbox backlog
with KEDA, typically using a few tens of megabytes of memory.

### Why Rust here, and not in Beacon.Api

`beacon-relay` is a narrow, I/O-heavy component where Rust's strengths (low memory per connection,
predictable latency, compile-time concurrency safety) matter, and its interface to the rest of the system
is a database table and HTTP. A .NET worker (Book III, Chapter 8) would also work well; the choice is a
legitimate trade-off, and for a team without Rust experience, the .NET version would be the pragmatic
default. Choosing Rust for one component like this is a low-risk way to gain experience.

---

## 7. What can go wrong

- **Blocking in async code** (`std::thread::sleep`, synchronous I/O, heavy CPU) stalling the runtime: use
  `spawn_blocking` or Rayon for CPU work.
- **Holding a `std::sync::Mutex` guard across `.await`** (the compiler often flags it because the guard isn't
  `Send`; using `tokio::sync::Mutex` makes it compile but can still hurt throughput).
- **Unbounded channels and unbounded spawning** under load.
- **Forgetting that dropped futures cancel**, so work stops halfway (design idempotent steps).
- **Mixing runtimes** or calling `block_on` inside async code.
- **SSRF** in any service that fetches user-supplied URLs.
- **Assuming "no data races" means "no concurrency bugs"**: deadlocks and logical races remain possible.

---

## 8. How an experienced engineer thinks about this

- **Let the type system carry the concurrency proof**, then still design to avoid shared state.
- **Data parallelism with Rayon; I/O concurrency with Tokio**: different tools for different problems.
- **Bound everything**: queues, concurrency, retries, timeouts.
- **Ownership-based cleanup** (permits, guards, connections) removes a class of leaks.
- **Use Rust where its strengths pay off**, at clear boundaries with the rest of the system.

---

## 9. Check yourself

**Questions**

1. What do `Send` and `Sync` mean? Why isn't `Rc<T>` `Send`?
2. What kind of concurrency bugs does Rust prevent at compile time, and which does it not?
3. How does Rayon parallelize `beacon-logscan`, and why does merging histograms matter?
4. How do Rust futures differ from C# tasks (starting, runtime, cancellation)?
5. Why must spawned tasks be `Send + 'static`?
6. How does `beacon-relay` apply back-pressure and bulkheads?
7. How does the relay defend against SSRF?

**Exercises**

1. Build `beacon-logscan` with Rayon and measure the speedup on several large generated log files.
2. Build `beacon-relay`, run a local receiver that randomly returns 500/429/200, and observe retries and
   per-endpoint limits.
3. Replace the HTTP intake with an outbox poller using `sqlx` and `FOR UPDATE SKIP LOCKED` (Book IV,
   Chapter 5).
4. Add the SSRF check: resolve the endpoint host and reject private and link-local IP ranges before each
   delivery, with tests.

**Interview-style questions**

- "What does 'fearless concurrency' mean in Rust?"
- "How does async work in Rust compared with C#?"
- "How would you build a reliable webhook delivery system?"

---

## 10. Going deeper

- [*The Rust Programming Language*, chapter 16 (concurrency) and 17 (async)](https://doc.rust-lang.org/book/)
- [Tokio tutorial](https://tokio.rs/tokio/tutorial) and [axum documentation](https://docs.rs/axum)
- Mara Bos, *Rust Atomics and Locks* (free online) — low-level concurrency, clearly explained.
- [Rayon documentation](https://docs.rs/rayon)

---

## Book XII wrap-up

You've seen a different answer to the questions Book I asked about memory, types, errors and concurrency:
ownership and borrowing instead of a garbage collector, enums and exhaustive matching instead of nulls and
flags, `Result` instead of exceptions, and compile-time prevention of data races. Beacon gained two small
Rust components where those properties pay off: a fast, parallel log scanner and a highly concurrent webhook
relay, each with a narrow boundary to the .NET system.

**Next:** [Book XIII — Architecture, Security & System Design](../13-architecture/README.md) steps back to
the design of whole systems, and how experienced engineers make those decisions.
