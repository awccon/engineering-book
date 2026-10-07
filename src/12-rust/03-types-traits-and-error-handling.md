# Types, Traits and Error Handling

Chapter 2 covered how Rust manages memory. This chapter covers how Rust models **data**,
**behavior** and **failure**: structs and enums (with pattern matching), traits and generics,
and `Result`-based error handling. Together they make Rust code very explicit: what data can
exist, what operations it supports, and every way an operation can fail, all checked by the
compiler.

Much of this will feel like a stricter, more consistent version of ideas from earlier books:
records and discriminated unions (Book I, Chapter 2; Book V, Chapter 4), interfaces and
generics (Book I, Chapters 3–4), and result types (Book I, Chapter 8).

> The code in this chapter is part of `beacon-logscan` and has been compiled and tested with
> stable Rust.

---

## 1. The problem: making illegal states and unhandled failures impossible

Two families of bugs this chapter targets:

- **Invalid states**: a ticket that's "Resolved" with no resolution time; a response that's both
  "loading" and "has data"; a nullable field nobody checks.
- **Unhandled failures**: an exception nobody catches until production; an error code silently
  ignored; a function whose signature doesn't reveal it can fail.

Rust addresses both with its type system: **enums** that carry data, **exhaustive matching**, no null,
and errors as **values in return types**.

---

## 2. Structs and enums

### Structs

```rust
#[derive(Debug, Clone, PartialEq)]
pub struct Ticket {
    pub id: u32,
    pub title: String,
    pub priority: Priority,
    pub status: Status,
}

struct Millis(f64);          // tuple struct: a newtype (like a C# record struct TicketId(int))
struct Marker;               // unit struct
```

`#[derive(...)]` generates trait implementations (`Debug` for printing, `Clone`, `PartialEq` for `==`,
`Hash`, `Default`, `serde::Serialize`...), the way C# records generate equality members.

### Enums carry data

Rust enums are **algebraic data types**: each variant can hold different data, like TypeScript
discriminated unions (Book V, Chapter 4):

```rust
pub enum Status {
    Open,
    InProgress { assignee: String },
    Resolved { assignee: String, resolved_at: std::time::SystemTime },
    Closed,
}
```

With this definition, "Resolved without a resolution time" or "InProgress without an assignee" **can't
be represented**. Compare Beacon's C# `Ticket`, which needed private setters, guard clauses and a
database check constraint (Book IV, Chapter 1) to protect the same invariants.

### `Option` and `Result` are just enums

```rust
enum Option<T> { Some(T), None }
enum Result<T, E> { Ok(T), Err(E) }
```

No null: a value that may be absent is `Option<T>`, and the compiler won't let you use it as a `T`
without handling `None`.

### Pattern matching

`match` must be **exhaustive**: every possible case handled, or it doesn't compile.

```rust
fn describe(status: &Status) -> String {
    match status {
        Status::Open => "open".to_string(),
        Status::InProgress { assignee } => format!("in progress ({assignee})"),
        Status::Resolved { assignee, .. } => format!("resolved by {assignee}"),
        Status::Closed => "closed".to_string(),
    }
}
```

Add a variant to `Status`, and every `match` that doesn't handle it becomes a compile error: the same
guarantee as `assertNever` in TypeScript or the exhaustive `switch` expressions in C#, but universal.

Matching works on ranges and conditions too. From `beacon-logscan`:

```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq, Hash)]
pub enum StatusClass { Success, Redirect, ClientError, ServerError, Other }

impl From<u16> for StatusClass {
    fn from(status: u16) -> Self {
        match status {
            200..=299 => StatusClass::Success,
            300..=399 => StatusClass::Redirect,
            400..=499 => StatusClass::ClientError,
            500..=599 => StatusClass::ServerError,
            _ => StatusClass::Other,
        }
    }
}
```

Other forms: `if let Some(x) = opt { ... }`, `let Some(x) = opt else { return ... };`, `matches!(v, Pattern)`.

---

## 3. Traits

A **trait** defines behavior a type can implement, like a C# interface, with important differences:

```rust
/// Something that can summarize a set of latency samples.
pub trait LatencySummary {
    fn record(&mut self, value_ms: f64);
    fn count(&self) -> u64;
    fn percentile(&self, p: f64) -> Option<f64>;

    /// Default method: available to every implementor.
    fn p95(&self) -> Option<f64> {
        self.percentile(95.0)
    }
}
```

- **Implementations are separate from type definitions**: `impl LatencySummary for HistogramSummary { ... }`.
  You can implement your traits for types you didn't write (including standard types), and standard
  traits for your types (with the "orphan rule": either the trait or the type must be yours).
- **Default methods** (like C# default interface methods; Book I, Chapter 3).
- **Associated types and constants**: `Iterator` has `type Item;`.
- **Standard traits** wire types into the language: `Display` (for `{}` formatting), `Debug` (`{:?}`),
  `Clone`, `Copy`, `PartialEq`/`Eq`, `PartialOrd`/`Ord`, `Hash`, `Default`, `From`/`Into` (conversions),
  `Iterator`, `Drop`, `Send`/`Sync` (thread safety; Chapter 4).

Two implementations for Beacon's latency statistics: exact (keeps every sample) and a fixed-memory
histogram (approximate):

```rust
/// Exact percentiles: keeps every sample (memory grows with input).
#[derive(Debug, Default)]
pub struct ExactSummary {
    samples: Vec<f64>,
}

impl LatencySummary for ExactSummary {
    fn record(&mut self, value_ms: f64) { self.samples.push(value_ms); }
    fn count(&self) -> u64 { self.samples.len() as u64 }
    fn percentile(&self, p: f64) -> Option<f64> {
        if self.samples.is_empty() { return None; }
        let mut sorted = self.samples.clone();
        sorted.sort_by(f64::total_cmp);
        let rank = ((p / 100.0) * (sorted.len() - 1) as f64).round() as usize;
        sorted.get(rank).copied()
    }
}

/// Fixed-memory histogram with logarithmic buckets (approximate percentiles).
#[derive(Debug)]
pub struct HistogramSummary {
    buckets: [u64; 64],
    count: u64,
}

impl Default for HistogramSummary {
    fn default() -> Self { Self { buckets: [0; 64], count: 0 } }
}

impl HistogramSummary {
    fn bucket_of(value_ms: f64) -> usize {
        // bucket i covers [2^(i/4), 2^((i+1)/4)) ms: roughly 19% relative error
        ((value_ms.max(1.0).log2() * 4.0) as usize).min(63)
    }
    fn upper_bound(bucket: usize) -> f64 { 2f64.powf((bucket + 1) as f64 / 4.0) }
}

impl LatencySummary for HistogramSummary {
    fn record(&mut self, value_ms: f64) {
        self.buckets[Self::bucket_of(value_ms)] += 1;
        self.count += 1;
    }
    fn count(&self) -> u64 { self.count }
    fn percentile(&self, p: f64) -> Option<f64> {
        if self.count == 0 { return None; }
        let target = ((p / 100.0) * self.count as f64).ceil().max(1.0) as u64;
        let mut seen = 0;
        for (i, n) in self.buckets.iter().enumerate() {
            seen += n;
            if seen >= target { return Some(Self::upper_bound(i)); }
        }
        None
    }
}
```

Note `f64::total_cmp`: floating-point numbers aren't totally ordered (`NaN`), so Rust doesn't let you sort
`f64`s with the default comparison; you must choose an ordering explicitly. The kind of edge case C# lets
you ignore.

---

## 4. Generics and dispatch

### Static dispatch (monomorphization)

```rust
pub fn scan_file<S: LatencySummary + Default>(path: &Path, max_bad_pct: u8) -> Result<FileReport<S>, ScanError> { ... }

let report = scan_file::<HistogramSummary>(path, 1)?;
```

`S: LatencySummary + Default` is a **trait bound** (like `where S : ILatencySummary, new()` in C#). The
compiler generates a specialized copy of `scan_file` for each concrete `S` used (**monomorphization**), so
calls are direct and inlinable, with zero run-time overhead. C# does this only for value types (Book I,
Chapter 4); Rust does it for everything.

`impl Trait` is shorthand in argument and return positions:

```rust
fn slowest(routes: impl Iterator<Item = (String, f64)>) -> Option<(String, f64)> {
    routes.max_by(|a, b| a.1.total_cmp(&b.1))
}
```

### Dynamic dispatch (trait objects)

When you need a collection of different types behind one trait, or to choose an implementation at run
time, use a **trait object** with a vtable, like C# interface references:

```rust
let summary: Box<dyn LatencySummary> = if exact { Box::new(ExactSummary::default()) }
                                       else { Box::new(HistogramSummary::default()) };
```

| | Static (`T: Trait`, `impl Trait`) | Dynamic (`dyn Trait`) |
|---|---|---|
| Resolved | Compile time | Run time (vtable) |
| Performance | Inlinable, zero overhead | Indirect call |
| Code size | One copy per type | One copy |
| Heterogeneous collections | No | Yes |

Default to static dispatch; use `dyn` when you need run-time flexibility.

---

## 5. Error handling

### Two kinds of failure

| Kind | Rust mechanism | C# analogy (Book I, Chapter 8) |
|---|---|---|
| **Recoverable, expected** (file missing, bad input, network error) | `Result<T, E>` | `Result<T>` / `TryX` / handled exceptions |
| **Bugs, broken invariants** (index out of bounds, `unwrap` on `None`) | `panic!` (unwinds or aborts the thread) | Unhandled exceptions from bugs |

Rust has **no exceptions** for expected failures. If a function can fail, its signature says so, and the
caller must deal with it.

### The `?` operator

```rust
fn read_config(path: &Path) -> Result<Config, ConfigError> {
    let text = std::fs::read_to_string(path)?;          // on Err: convert and return early
    let config: Config = toml::from_str(&text)?;
    Ok(config)
}
```

`?` returns early with the error (converted via `From` into the function's error type), which gives
exception-like brevity with explicit, typed propagation.

### Library errors with `thiserror`

Libraries define **specific error enums** so callers can match on what went wrong:

```rust
use std::path::PathBuf;
use thiserror::Error;

#[derive(Debug, Error)]
pub enum ScanError {
    #[error("cannot open {path}")]
    Open { path: PathBuf, #[source] source: std::io::Error },

    #[error("I/O error while reading {path}")]
    Read { path: PathBuf, #[source] source: std::io::Error },

    #[error("{bad} of {total} lines in {path} were malformed (limit {limit}%)")]
    TooManyBadLines { path: PathBuf, bad: u64, total: u64, limit: u8 },
}
```

`#[source]` chains the underlying error (like `InnerException`), and the message includes the path:
context the person debugging needs (Book I, Chapter 8's "write messages for 3 a.m.").

```rust
pub fn scan_file<S: LatencySummary + Default>(path: &Path, max_bad_pct: u8) -> Result<FileReport<S>, ScanError> {
    let file = File::open(path).map_err(|source| ScanError::Open { path: path.to_owned(), source })?;
    let mut reader = BufReader::new(file);
    let mut line = String::new();
    let mut report = FileReport::<S>::default();

    loop {
        line.clear();
        let n = reader.read_line(&mut line).map_err(|source| ScanError::Read { path: path.to_owned(), source })?;
        if n == 0 { break; }
        report.total += 1;
        match serde_json::from_str::<AccessRecord>(&line) {
            Ok(rec) => {
                if StatusClass::from(rec.status) == StatusClass::ServerError { report.server_errors += 1; }
                report.latency.record(rec.duration_ms);
            }
            Err(_) => report.bad += 1,
        }
    }

    if report.total > 0 && report.bad * 100 > report.total * u64::from(max_bad_pct) {
        return Err(ScanError::TooManyBadLines {
            path: path.to_owned(), bad: report.bad, total: report.total, limit: max_bad_pct,
        });
    }
    Ok(report)
}
```

Malformed lines are **expected** (logs get truncated), so they're counted, not fatal, until they exceed a
threshold, which signals a real problem (wrong file format). That's Book I, Chapter 8's classification of
failures, encoded in types.

### Application errors with `anyhow`

Applications (the binary's `main`) usually just need to report errors with context, not match on them.
`anyhow::Result` accepts any error type and adds context:

```rust
use anyhow::{Context, Result};

fn main() -> Result<()> {
    let args = Args::parse();
    for path in &args.files {
        let report = scan::scan_file::<HistogramSummary>(path, args.max_bad_pct)?;
        println!("{}: {} lines, {} 5xx, p95 ≈ {:.0} ms",
                 path.display(), report.total, report.server_errors, report.latency.p95().unwrap_or(0.0));
    }
    Ok(())
}
```

Returning an `Err` from `main` prints the error chain and exits with a non-zero code, the right CLI behavior
(Book I, Chapter 8; Book X, Chapter 3):

```text
$ beacon-logscan truncated.jsonl
Error: 1 of 3 lines in truncated.jsonl were malformed (limit 1%)
$ echo $?
1
```

Rule of thumb: **`thiserror` in libraries, `anyhow` in applications.**

### `unwrap` and `expect`

`unwrap()` turns a `None`/`Err` into a panic. Fine in tests, prototypes, and when an invariant guarantees
success (with `expect("reason")` documenting why). In production paths for expected failures, it's the
equivalent of an unhandled exception. `cargo clippy` can be configured to flag `unwrap` in non-test code.

---

## 6. Testing in Rust

Tests live next to the code, in a `#[cfg(test)]` module compiled only for `cargo test`:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn fill<S: LatencySummary + Default>() -> S {
        let mut s = S::default();
        for v in 1..=100 { s.record(v as f64); }
        s
    }

    #[test]
    fn exact_p95() {
        let s: ExactSummary = fill();
        assert_eq!(s.p95(), Some(95.0));
    }

    #[test]
    fn histogram_p95_is_close() {
        let s: HistogramSummary = fill();
        let p95 = s.p95().unwrap();
        assert!((90.0..=120.0).contains(&p95), "p95 was {p95}");
    }

    #[test]
    fn empty_has_no_percentile() {
        assert_eq!(ExactSummary::default().p95(), None);
    }
}
```

```text
$ cargo test
running 3 tests
test result: ok. 3 passed; 0 failed
```

The generic `fill` helper runs the same scenario against both implementations, a compact form of the
"same contract, several implementations" tests from Book I, Chapter 14. Integration tests go in `tests/`;
code examples in doc comments run as **doc tests**, so documentation can't go stale.

---

## 7. In practice: `beacon-logscan`'s structure

```text
beacon-logscan/
  Cargo.toml
  src/
    main.rs          CLI (clap), application errors (anyhow)
    record.rs        AccessRecord<'a> (zero-copy deserialization, Chapter 2)
    scan.rs          scan_file<S: LatencySummary>, StatusClass
    percentiles.rs   LatencySummary trait, ExactSummary, HistogramSummary, tests
    error.rs         ScanError (thiserror)
```

Design decisions, mapped to this chapter:

- **A trait for the summary strategy**, chosen at compile time (`HistogramSummary` by default for
  constant memory; `ExactSummary` available for small files), with static dispatch in the hot loop.
- **Enums for classification** with exhaustive matching.
- **Typed library errors** with paths and sources; `anyhow` at the top.
- **Expected failures as data** (malformed lines counted) with a threshold that turns them into an error.
- **Tests per implementation through the trait.**

Chapter 4 makes it parallel across CPU cores and builds the async `beacon-relay` service.

---

## 8. What can go wrong

- **`unwrap()` everywhere** in production code, turning expected failures into crashes.
- **Stringly-typed errors** (`Result<T, String>`) that callers can't handle specifically.
- **Errors without context** ("No such file or directory" without the path).
- **Overusing trait objects** (`Box<dyn ...>`) where generics would be simpler and faster.
- **Complex generic bounds** that make signatures unreadable (introduce named traits or type aliases).
- **Wildcard `_` arms in `match`** that silently swallow new enum variants; prefer listing variants where
  new cases deserve attention.

---

## 9. How an experienced engineer thinks about this

- **Model the domain with enums** so invalid states can't exist.
- **Every failure in the signature**: `Result` for expected failures, panics only for bugs.
- **Traits describe capabilities**; generics give zero-cost abstraction; `dyn` when you need run-time choice.
- **Context in errors** is a feature for the person debugging.
- **Let the compiler enforce exhaustiveness** as the code evolves.

---

## 10. Check yourself

**Questions**

1. How do Rust enums differ from C# enums? Give an example of an invalid state they prevent.
2. Why must `match` be exhaustive, and why is that valuable over time?
3. How do traits differ from C# interfaces?
4. Compare static and dynamic dispatch. When would you use each?
5. What does the `?` operator do?
6. When should you use `thiserror` vs `anyhow`?
7. When is `unwrap` acceptable?

**Exercises**

1. Model Beacon's ticket status as the data-carrying `Status` enum and write a `resolve` function that
   only accepts `InProgress` tickets (returning a `Result`).
2. Add a `Display` implementation for `StatusClass` and use it in output.
3. Add a third `LatencySummary` implementation (e.g. reservoir sampling) and run the shared tests against it.
4. Add an integration test in `tests/` that scans a fixture file and checks the report.

**Interview-style questions**

- "How does error handling in Rust differ from exceptions?"
- "What are traits? How are they different from interfaces?"
- "What's monomorphization?"

---

## 11. Going deeper

- [*The Rust Programming Language*, chapters 6, 9, 10 and 18](https://doc.rust-lang.org/book/)
- [thiserror](https://docs.rs/thiserror) and [anyhow](https://docs.rs/anyhow) documentation.
- [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/)

**Next:** [Chapter 4 — Concurrency, Async and Networking](04-concurrency-async-and-networking.md) builds
fearless concurrency and an async network service.
