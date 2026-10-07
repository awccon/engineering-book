# Why Rust, and When

> **🔄 Current (as of October 2026):** Rust releases a new stable version every six weeks; the
> current edition is **Rust 2024**. Examples use stable Rust with Cargo. Install with `rustup`.

Book I began by asking what the .NET runtime does for you and what it costs. Rust is the
answer to a different question: **what if you could have memory safety without a garbage
collector, and concurrency without data races, enforced at compile time?** Rust delivers
that, at the cost of a steeper learning curve and a compiler that rejects programs other
languages would accept.

This book (Book XII) isn't about becoming a Rust expert. It's about understanding Rust's ideas
well enough to read Rust code, write small tools and services, judge when Rust is the right
choice, and become a better C# developer by seeing memory and ownership made explicit.

---

## 1. The problem: the safety–performance trade-off

For decades, systems languages forced a choice (Book I, Chapter 11):

| | C / C++ | C#, Java, Go | Rust |
|---|---|---|---|
| Memory management | Manual | Garbage collector | **Ownership, checked at compile time** |
| Memory safety | ✗ (use-after-free, buffer overflows, double free) | ✓ | ✓ |
| Data races | Possible | Possible (Book I, Chapter 10) | **Prevented at compile time** (in safe Rust) |
| Runtime overhead | Minimal | GC, JIT, runtime | **Minimal** (no GC, no runtime) |
| Predictable latency | ✓ | GC pauses (usually small) | ✓ |
| Learning curve | High (and unsafe) | Moderate | **High** (but safe) |

Memory-safety bugs in C and C++ are the cause of a large share of serious security
vulnerabilities in browsers, operating systems and infrastructure. Rust removes that whole class
of bugs from safe code, while matching C/C++ performance. That's why Rust is now used in the Linux
and Windows kernels, browsers, cloud infrastructure, databases and developer tooling (the uv and
Ruff Python tools from Book X, Vite's new bundler, and many CLI tools are written in Rust).

---

## 2. The mental model: the compiler as a strict reviewer

Rust's central idea: every value has exactly one **owner**; when the owner goes out of scope, the
value is freed. References (**borrows**) let code use a value without owning it, under rules the
compiler checks:

- either **one mutable reference** or **any number of shared (read-only) references**, never both
  at once;
- references must never **outlive** the value they point to.

The compiler's **borrow checker** enforces these rules. Programs that would have memory or data-race
bugs **don't compile**. Chapter 2 covers this in depth.

```text
 C#:   allocate → many references anywhere → GC finds unreachable objects later → free
 Rust: allocate → one owner (+ checked borrows) → owner's scope ends → free immediately (deterministic)
```

> **🧱 Durable:** Rust makes **ownership** explicit. Every language has ownership questions ("who
> is responsible for this resource? who may change it? how long must it live?"); C# answers them
> at run time with the GC and leaves resources (Book I, Chapter 11's `IDisposable`) and thread
> safety to discipline. Rust answers them at compile time.

---

## 3. A first look

```rust
// src/main.rs
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
enum Priority {
    Low,
    Normal,
    High,
    Urgent,
}

#[derive(Debug)]
struct Ticket {
    id: u32,
    title: String,
    priority: Priority,
    assignee: Option<String>,
}

impl Ticket {
    fn is_unassigned_urgent(&self) -> bool {
        self.priority == Priority::Urgent && self.assignee.is_none()
    }
}

fn main() {
    let tickets = vec![
        Ticket { id: 1, title: "VPN drops".into(), priority: Priority::Urgent, assignee: None },
        Ticket { id: 2, title: "Printer offline".into(), priority: Priority::Normal, assignee: Some("maria".into()) },
    ];

    let unassigned: Vec<&Ticket> = tickets.iter().filter(|t| t.is_unassigned_urgent()).collect();
    for t in &unassigned {
        println!("T-{} {} ({:?})", t.id, t.title, t.priority);
    }
}
```

```bash
cargo new beacon-logscan && cd beacon-logscan
cargo run
```

Familiar pieces for a C# developer: structs, enums, methods in `impl` blocks, iterators with
closures (like LINQ), `Option<T>` (like nullable types, but enforced), `Vec<T>` (like `List<T>`).
Unfamiliar pieces: `&self` and `&Ticket` (borrows), `.into()` conversions, `#[derive(...)]`
generating trait implementations, no `null`, no exceptions, no classes or inheritance.

---

## 4. Rust vs C#: a map

| Concept | C# | Rust |
|---|---|---|
| Value types / reference types | `struct` / `class` | Everything is a value; heap via `Box`, `Vec`, `String`, `Rc`, `Arc` |
| Memory | GC | Ownership + borrowing; deterministic drop |
| `null` | `null` + nullable reference types | **No null**; `Option<T>` (`Some`/`None`) |
| Errors | Exceptions | **`Result<T, E>`** values + `?` operator; `panic!` for bugs |
| Interfaces | `interface` | **Traits** (also with generics, static dispatch, default methods) |
| Inheritance | Classes | None; composition + traits |
| Generics | Reified, JIT-specialized | Monomorphized at compile time (like C++ templates, but checked) |
| Pattern matching | `switch` expressions | `match` (exhaustive, central to the language) |
| Enums | Named integers | **Algebraic data types**: each variant can carry data (like discriminated unions) |
| Immutability | Opt-in (`readonly`, records) | **Default**; `mut` to opt in |
| Async | `Task`, built-in runtime | `Future`, **runtime chosen by you** (Tokio) |
| Packages / build | NuGet / MSBuild / `dotnet` | crates.io / **Cargo** |
| Disposal | `IDisposable` + `using` | `Drop` trait, automatic at scope end (RAII) |
| Reflection | Rich | Minimal; macros generate code at compile time instead |

Many of Book I's lessons reappear in Rust as compiler rules: immutability by default (Chapter 2),
results instead of exceptions for expected failures (Chapter 8), discriminated unions making
illegal states unrepresentable (Book V, Chapter 4), no shared mutable state across threads
without synchronization (Chapter 10), deterministic resource cleanup (Chapter 11).

---

## 5. Cargo and the ecosystem

**Cargo** is Rust's build tool and package manager in one, the most cohesive toolchain covered in
this book:

```bash
cargo new my-app           # new binary project (cargo new --lib for a library)
cargo build                # debug build
cargo build --release      # optimized build (compile times are longer than C#)
cargo run -- --input x     # build and run with arguments
cargo test                 # unit, integration and doc tests
cargo fmt                  # format (rustfmt)
cargo clippy               # lints (excellent, opinionated)
cargo add serde --features derive   # add a dependency
cargo doc --open           # generate and open documentation
```

```toml
# Cargo.toml
[package]
name = "beacon-logscan"
version = "0.1.0"
edition = "2024"

[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
clap = { version = "4", features = ["derive"] }
anyhow = "1"
```

Packages are **crates**, published on **crates.io**; `Cargo.lock` locks exact versions (commit it for
applications). Widely used crates you'll meet: `serde` (serialization), `tokio` (async runtime),
`axum` (web framework), `reqwest` (HTTP client), `sqlx` (async SQL), `clap` (CLI arguments),
`anyhow`/`thiserror` (errors), `tracing` (structured logging and spans), `rayon` (data parallelism).

---

## 6. When Rust makes sense

Rust's costs are real: a steep learning curve, slower compile times, more upfront design, a smaller
hiring pool than C#, and fewer batteries-included frameworks for typical business applications.

| Good fit | Why |
|---|---|
| **Performance-critical services and hot paths** | C-level speed, no GC pauses, low memory |
| **Infrastructure and systems software** | Proxies, databases, storage engines, runtimes, OS components |
| **CLI tools and developer tooling** | Single static binary, instant startup, fast |
| **Resource-constrained environments** | Embedded, IoT, edge, small containers, WebAssembly |
| **Security-sensitive parsing** | Parsers for untrusted input (file formats, network protocols) without memory bugs |
| **Libraries used from other languages** | Fast native extensions for Python (PyO3), Node, .NET (via C ABI) |
| **Highly concurrent network services** | Async + compile-time data-race freedom |

> **🧭 When not to use Rust:** For typical line-of-business applications (CRUD APIs, admin
> portals, workflows like most of Beacon), C#/.NET offers higher productivity, a richer ecosystem
> for business needs (EF Core, ASP.NET Core Identity, mature tooling) and performance that's more
> than sufficient. Choose Rust for a component when you have a concrete need (latency, memory,
> safety of native code, deployment footprint), not to rewrite working .NET services. "Rewrite it in
> Rust" is rarely a business case by itself.

### Rust and .NET together

The practical pattern for a .NET team: keep the application in .NET, and use Rust for a **specific
component** where its strengths matter, connected via:

- a **network boundary** (HTTP/gRPC service, or a queue consumer): simplest and most common;
- a **native library** called from .NET via P/Invoke (`[LibraryImport]`, Book I, Chapter 12) over a C
  ABI: lowest latency, more complex (unsafe boundary, memory ownership across languages);
- **WebAssembly** components in some scenarios.

---

## 7. In practice: Beacon's Rust components

Two components in Book XII, chosen because they fit Rust's strengths:

1. **`beacon-logscan`** (Chapters 1–3): a CLI that scans large JSONL audit/access log exports
   (gigabytes) and produces statistics: requests per tenant, error rates, slowest endpoints. It
   streams files with constant memory, runs in parallel across cores, and ships as a single binary to
   on-call engineers.
2. **`beacon-relay`** (Chapter 4): a small, highly concurrent service that delivers webhooks to
   customers' endpoints (a new Beacon feature: "notify my systems when a ticket changes"), with
   retries, back-off, per-endpoint concurrency limits and signatures. Beacon.Api writes webhook
   events to the outbox; `beacon-relay` consumes and delivers them. Thousands of slow customer
   endpoints shouldn't tie up the main API, and a small memory footprint keeps costs low.

Both have narrow, well-defined responsibilities and communicate with the .NET system through files,
HTTP or the database, which keeps the Rust surface small and the team's main expertise in .NET.

Start `beacon-logscan`:

```bash
cargo new beacon-logscan
cd beacon-logscan
cargo add serde --features derive
cargo add serde_json clap --features clap/derive
cargo add anyhow
cargo run -- --help
```

---

## 8. What can go wrong

- **Choosing Rust for prestige** rather than a concrete need, then paying the learning and hiring costs.
- **Fighting the borrow checker** by cloning everything or wrapping everything in `Rc<RefCell<…>>`,
  losing Rust's benefits (Chapter 2 explains better approaches).
- **Rewriting stable systems** instead of targeting the component that needs it.
- **Underestimating compile times and tooling differences** in CI.
- **Unsafe code at FFI boundaries** written carelessly.
- **Too many small crates** pulled in without review (the same supply-chain concerns as npm and PyPI).

---

## 9. How an experienced engineer thinks about this

- **Rust trades learning and compile-time effort for run-time safety and performance.**
- **Use it surgically**: specific components with clear boundaries and clear benefits.
- **Learn it for its ideas** even if you rarely ship it: ownership, explicit errors, exhaustive matching
  and data-race freedom make you a better C# engineer.
- **Measure the need**: latency, memory, throughput, footprint, safety.

---

## 10. Check yourself

**Questions**

1. What trade-off does Rust remove, and how?
2. What are Rust's two borrowing rules?
3. How does Rust replace `null` and exceptions?
4. Compare Rust enums with C# enums.
5. What does Cargo provide?
6. Name four good fits and two poor fits for Rust.
7. How can Rust and .NET components work together?

**Exercises**

1. Install Rust with `rustup`, create `beacon-logscan`, and run the example from section 3.
2. Add a `match` on `Priority` that returns the SLA target as a `std::time::Duration`, and see what
   happens when you leave out a variant.
3. Run `cargo clippy` on your code and read every suggestion.
4. Write a one-page "should we use Rust for X?" analysis for a component in a system you know.

**Interview-style questions**

- "What makes Rust memory-safe without a garbage collector?"
- "When would you choose Rust over C# or Go?"
- "How would you integrate a Rust component into a .NET system?"

---

## 11. Going deeper

- [*The Rust Programming Language*](https://doc.rust-lang.org/book/) ("the Book," free) — start here.
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)
- [Rustlings](https://github.com/rust-lang/rustlings) — small exercises.
- [Microsoft: Rust for .NET developers / Learn modules](https://learn.microsoft.com/training/paths/rust-first-steps/)

**Next:** [Chapter 2 — Ownership, Borrowing and Lifetimes](02-ownership-borrowing-and-lifetimes.md)
explains the core idea that makes Rust different.
