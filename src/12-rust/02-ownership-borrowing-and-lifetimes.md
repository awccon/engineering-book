# Ownership, Borrowing and Lifetimes

Ownership is the idea that makes Rust Rust. It's also what every newcomer struggles with: the
borrow checker rejects code that looks perfectly reasonable to a C# developer. The struggle
fades once the mental model clicks, and the model maps directly onto things you already know
from Book I: value vs reference semantics, object lifetimes, who's responsible for disposing
resources, and why shared mutable state is dangerous.

This chapter builds that model step by step, with C# comparisons throughout.

---

## 1. The problem: what the garbage collector was doing for you

In C# (Book I, Chapters 2 and 11):

- Any number of references can point to an object, from anywhere, at any time.
- The GC frees the object when nothing references it anymore, at some later time.
- Resources (files, sockets) need `Dispose`, and forgetting it leaks them.
- Two threads can mutate the same object; preventing races is your discipline (Chapter 10).
- A mutation through one reference is visible through all others (accidental sharing bugs).

Rust has no GC, so it needs another way to know **when to free memory**, and it uses the same
mechanism to prevent **dangling references** and **data races**.

---

## 2. The mental model: ownership

Three rules:

1. **Each value has exactly one owner** (a variable, a struct field, a collection).
2. **When the owner goes out of scope, the value is dropped** (memory freed, resources released).
3. **Ownership can be moved** to another owner; the old owner can no longer use the value.

```rust
fn main() {
    let title = String::from("VPN drops");   // `title` owns a heap-allocated string
    let moved = title;                        // ownership MOVES to `moved`
    // println!("{title}");                   // ✗ compile error: value used after move
    println!("{moved}");
}                                             // `moved` goes out of scope → string freed here
```

```text
 stack                     heap
 ┌──────────────────┐      ┌─────────────────────┐
 │ title: ptr,len,cap├─✗   │ "VPN drops"          │
 │ moved: ptr,len,cap├────►│                      │
 └──────────────────┘      └─────────────────────┘
 After the move, only `moved` may use the string; when `moved` is dropped, the heap memory is freed once.
```

Compare with C#: `var moved = title;` copies a **reference**; both variables point to the same
string, and the GC frees it later. In Rust, a plain assignment of a heap-owning value **moves**
it, so there's always exactly one owner responsible for freeing it. No double frees, no leaks, no GC.

### Moves happen everywhere

```rust
fn print_title(t: String) { println!("{t}"); }   // takes ownership

let title = String::from("VPN drops");
print_title(title);          // moved into the function; dropped when the function returns
// print_title(title);       // ✗ already moved
```

### Copy types

Small, stack-only types (`i32`, `u64`, `f64`, `bool`, `char`, and structs/enums made only of `Copy`
types that `#[derive(Clone, Copy)]`) are **copied** instead of moved, like C# value types:

```rust
let a: u32 = 5;
let b = a;          // copy; both usable
```

### Clone: explicit deep copies

```rust
let title = String::from("VPN drops");
let copy = title.clone();      // explicit heap allocation + copy; both owned separately
```

Cloning is always **visible** in Rust code. That's deliberate: in C#, copies (or their absence) are
often invisible.

### Drop: deterministic cleanup

When an owner goes out of scope, Rust calls `drop`, freeing memory and releasing resources:

```rust
{
    let file = std::fs::File::open("audit.jsonl")?;   // opened
    // ... use file ...
}                                                      // closed here, automatically
```

This is **RAII** (Resource Acquisition Is Initialization), like C#'s `using` but applied to **every**
value automatically, without a keyword to forget.

---

## 3. Borrowing: using without owning

Moving ownership into every function would be impractical. **References** borrow a value without
taking ownership:

```rust
fn title_len(t: &String) -> usize { t.len() }    // borrows; doesn't take ownership

let title = String::from("VPN drops");
let n = title_len(&title);                        // lend it
println!("{title} has {n} bytes");                // still ours
```

Two kinds of references:

| Reference | Syntax | Allows | How many at once |
|---|---|---|---|
| **Shared** | `&T` | Read | Any number |
| **Mutable (exclusive)** | `&mut T` | Read and write | **Exactly one**, and no shared references at the same time |

```rust
let mut tickets = vec![1, 2, 3];

let first = &tickets[0];       // shared borrow
tickets.push(4);               // ✗ compile error: can't mutate while `first` is borrowed
println!("{first}");
```

Why is this an error? `push` may need to reallocate the vector's buffer, moving the elements, and
`first` would point to freed memory: a **dangling pointer**, exactly the kind of bug that causes
security vulnerabilities in C++. C# prevents memory corruption here differently (references to the
list, not into its buffer) but has its own version: modifying a collection while enumerating it
throws `InvalidOperationException` at run time (Book I, Chapter 5). Rust catches it at compile time.

> **🧱 Durable:** "**Aliasing XOR mutability**": data can be shared, or mutated, but not both at the
> same time. This one rule prevents dangling references, iterator invalidation and data races. It's
> worth remembering even in C#: most concurrency and accidental-sharing bugs violate it.

### Non-lexical lifetimes

Borrows last only as long as they're **used**, not until the end of the block:

```rust
let mut tickets = vec![1, 2, 3];
let first = &tickets[0];
println!("{first}");           // last use of `first`: the borrow ends here
tickets.push(4);               // ✓ fine
```

The compiler reasons precisely about where each reference is used.

### Borrowing in methods

```rust
impl Ticket {
    fn title(&self) -> &str { &self.title }                  // shared borrow of self
    fn assign_to(&mut self, who: String) { self.assignee = Some(who); }   // exclusive borrow
    fn into_summary(self) -> TicketSummary { /* consumes the ticket */ todo!() }   // takes ownership
}
```

The method signature tells you exactly what it does with the value: read it, mutate it, or consume it.
In C#, all three look the same (`ticket.Method()`).

### Slices: borrowed views

`&str` is a borrowed view of string data (like `ReadOnlySpan<char>`, Book I, Chapter 5), and `&[T]` a
view of a contiguous sequence (like `ReadOnlySpan<T>`). Functions should usually accept slices:

```rust
fn count_words(text: &str) -> usize { text.split_whitespace().count() }

count_words("literal");               // &'static str
count_words(&owned_string);           // &String coerces to &str
count_words(&owned_string[0..10]);    // a slice of it
```

---

## 4. Lifetimes: how long references are valid

A **lifetime** is the region of code during which a reference is valid. The compiler infers most of
them. You write them only when a function returns a reference and the compiler can't tell which input
it borrows from:

```rust
// Which input does the returned reference come from? The compiler needs to know
// so callers don't use the result after that input is gone.
fn longer<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() >= b.len() { a } else { b }
}
```

`'a` is a **lifetime parameter**: "the returned reference lives at most as long as both inputs." It's
not a runtime thing; it's a constraint the compiler checks:

```rust
let result;
{
    let short_lived = String::from("short");
    result = longer("a long string literal", &short_lived);
}                                   // short_lived dropped
// println!("{result}");            // ✗ compile error: `short_lived` does not live long enough
```

Lifetimes also appear in structs that hold references:

```rust
struct LogLine<'a> {
    tenant: &'a str,      // borrowed from the input buffer: no allocation per field
    path: &'a str,
    status: u16,
}
```

A `LogLine` can't outlive the buffer it borrows from. That's how Rust parsers achieve **zero-copy**
parsing safely, which in C# requires `ref struct`s and spans with their restrictions (Book I, Chapter 5:
`Span<T>` can't escape to the heap for the same underlying reason).

### Lifetime elision

Common patterns don't need annotations: a function with one reference input returning a reference gets
the input's lifetime; methods returning references get `&self`'s lifetime. You'll write explicit
lifetimes far less often than you might fear.

### `'static`

`'static` means "valid for the whole program": string literals, and owned data with no borrows. When a
thread or async task requires `'static`, it means "don't give me references to your stack; give me owned
data" (Chapter 4).

---

## 5. Shared ownership and interior mutability

Sometimes one owner isn't enough: a value genuinely shared by several parts of the program, like a
configuration or a cache. Rust provides explicit, opt-in tools:

| Type | Meaning | C# analogy |
|---|---|---|
| `Box<T>` | Single owner of a heap value | A reference with exactly one owner |
| `Rc<T>` | **Reference-counted** shared ownership, single thread | Shared reference (GC-like, by counting) |
| `Arc<T>` | **Atomically** reference-counted, thread-safe sharing | Shared reference across threads |
| `RefCell<T>` | Borrow rules checked **at run time** (panics if violated) | — |
| `Mutex<T>` / `RwLock<T>` | Mutation through shared references, with locking | `lock` around a field (Book I, Chapter 10) |
| `Cell<T>`, atomics | Simple interior mutability for `Copy` values / counters | `Interlocked` |

```rust
use std::sync::{Arc, Mutex};

let stats = Arc::new(Mutex::new(Stats::default()));   // shared, thread-safe, mutable through a lock
let s2 = Arc::clone(&stats);                          // cheap: increments a counter
std::thread::spawn(move || {
    s2.lock().unwrap().requests += 1;                  // the lock guard is dropped at end of statement
});
```

Notice: in Rust, a value shared across threads **must** be in a thread-safe container, or the program
doesn't compile. The `Mutex` **owns** the data it protects, so you can't access it without holding the
lock. In C#, a `lock` and the data it protects are separate, and nothing stops you from forgetting the lock.

> **⚠️ What can go wrong:** Newcomers often escape the borrow checker by wrapping everything in
> `Rc<RefCell<T>>` or `Arc<Mutex<T>>` and cloning freely. The code compiles, but you've rebuilt a slower,
> panicky version of a garbage-collected object graph. Usually, restructuring helps more: pass references
> down, return owned results up, use indices or IDs instead of references between objects, and keep
> ownership trees simple.

---

## 6. Common borrow-checker fights and fixes

| Error | Typical cause | Fix |
|---|---|---|
| "use of moved value" | Passed ownership somewhere, then used it again | Pass `&value`, or `.clone()` if you really need two copies |
| "cannot borrow as mutable because it is also borrowed as immutable" | Holding a reference while mutating the container | End the borrow first (shorter scopes), copy out the needed value, or restructure |
| "does not live long enough" | Returning or storing a reference to a local | Return an owned value (`String` instead of `&str`) |
| "cannot move out of borrowed content" | Taking ownership of a field through a reference | Clone, borrow instead, or use `std::mem::take`/`Option::take` |
| Two mutable borrows of a struct's fields | Calling a `&mut self` method while holding a field borrow | Borrow fields separately (`let a = &mut self.a; let b = &mut self.b;`), split the struct |

A reliable rule of thumb for beginners: **own data in structs, borrow in function parameters, return owned
data from functions**. Optimize with borrowed return values later, when you understand the lifetimes.

---

## 7. In practice: streaming log parsing in `beacon-logscan`

Beacon's access logs are JSONL (one JSON object per line), gigabytes per day. The tool must process them
in **constant memory** and **without allocating per field**.

```rust
// src/record.rs
use serde::Deserialize;

/// One access log entry, borrowing string fields from the line buffer (zero-copy).
#[derive(Debug, Deserialize)]
pub struct AccessRecord<'a> {
    #[serde(borrow)]
    pub tenant: &'a str,
    #[serde(borrow)]
    pub route: &'a str,
    pub status: u16,
    pub duration_ms: f64,
}
```

```rust
// src/stats.rs
use std::collections::HashMap;

#[derive(Debug, Default)]
pub struct RouteStats {
    pub count: u64,
    pub errors: u64,
    pub durations: Vec<f64>,       // for percentiles; sampled in Chapter 4 if memory matters
}

#[derive(Debug, Default)]
pub struct Stats {
    pub by_route: HashMap<String, RouteStats>,     // owned keys: outlive each line
    pub by_tenant: HashMap<String, u64>,
}

impl Stats {
    pub fn record(&mut self, r: &crate::record::AccessRecord<'_>) {
        let route = self.by_route.entry(r.route.to_owned()).or_default();
        route.count += 1;
        if r.status >= 500 { route.errors += 1; }
        route.durations.push(r.duration_ms);
        *self.by_tenant.entry(r.tenant.to_owned()).or_default() += 1;
    }
}
```

```rust
// src/main.rs
mod record;
mod stats;

use std::{fs::File, io::{BufRead, BufReader}, path::PathBuf};
use anyhow::{Context, Result};
use clap::Parser;

#[derive(Parser)]
#[command(about = "Scan Beacon access logs (JSONL) and print statistics")]
struct Args {
    /// Log files to scan
    files: Vec<PathBuf>,
}

fn main() -> Result<()> {
    let args = Args::parse();
    let mut stats = stats::Stats::default();
    let mut bad_lines = 0u64;

    for path in &args.files {
        let file = File::open(path).with_context(|| format!("opening {}", path.display()))?;
        let mut reader = BufReader::new(file);
        let mut line = String::new();                         // one reusable buffer

        while reader.read_line(&mut line)? > 0 {
            match serde_json::from_str::<record::AccessRecord>(&line) {
                Ok(rec) => stats.record(&rec),               // `rec` borrows from `line`...
                Err(_) => bad_lines += 1,
            }
            line.clear();                                     // ...so `rec` must be gone before we reuse `line`
        }
    }

    for (route, s) in &stats.by_route {
        println!("{route:40} {:>8} req {:>6.2}% 5xx", s.count, 100.0 * s.errors as f64 / s.count as f64);
    }
    eprintln!("skipped {bad_lines} malformed lines");
    Ok(())
}
```

What the borrow checker guarantees here:

- `AccessRecord` borrows `tenant` and `route` directly from `line`: no allocation per field.
  (One caveat: a JSON string containing escape sequences like `\"` can't be borrowed as-is, so
  deserialization fails for that line. Using `Cow<'a, str>` with `#[serde(borrow)]` borrows when possible
  and allocates only when needed.)
- The compiler **forces** `rec` to be dead before `line.clear()` reuses the buffer. If we tried to store
  `rec` in `stats` (keeping the borrow alive), it wouldn't compile; that's why `Stats` stores **owned**
  `String` keys via `to_owned()`. In C#, the equivalent span-based code relies on you remembering that the
  buffer will be overwritten.
- Files are closed automatically when `reader` goes out of scope.
- Memory use is bounded by the number of distinct routes and tenants (plus the durations vector, which
  Chapter 3 improves), not by file size.

```bash
cargo run --release -- /data/beacon-access-2026-10-06.jsonl
```

---

## 8. What can go wrong

- **Fighting the borrow checker** instead of restructuring ownership.
- **Cloning everywhere** to make errors go away, hurting performance and clarity.
- **`Rc<RefCell<T>>` object graphs** recreating GC-style designs with run-time panics.
- **Holding locks too long** (a `MutexGuard` kept alive across slow work), the same problem as in C#.
- **Over-annotating lifetimes** instead of returning owned values.
- **Reference cycles with `Rc`** leaking memory (use `Weak` for back-references).

---

## 9. How an experienced engineer thinks about this

- **Ownership is a design question**: who owns this data, who reads it, who changes it, how long it lives.
- **Aliasing XOR mutability** prevents a whole class of bugs, in any language.
- **Borrow in parameters, own in structs, return owned values**, then optimize.
- **Shared ownership and interior mutability are explicit tools**, used deliberately, not by default.
- **The borrow checker's errors are usually design feedback.**

---

## 10. Check yourself

**Questions**

1. State Rust's three ownership rules.
2. What's the difference between a move, a copy and a clone?
3. What are the two borrowing rules, and what bugs do they prevent?
4. Why can't you push to a `Vec` while holding a reference to one of its elements?
5. What is a lifetime parameter? Does it exist at run time?
6. When would you use `Arc<Mutex<T>>`? Why does the `Mutex` own the data?
7. How does `beacon-logscan` parse without allocating per field, and what does the compiler enforce?

**Exercises**

1. Write a function that returns the longest ticket title from a `&[Ticket]` as `&str`, with the correct
   lifetime.
2. Reproduce each error in section 6 deliberately, read the compiler's message, and fix it.
3. Change `AccessRecord` to own its strings (`String`) and measure the time difference on a large file.
4. Implement a struct holding a `HashMap<u32, Ticket>` with methods `get(&self)`, `assign(&mut self)` and
   `into_vec(self)`; explain what each signature promises.

**Interview-style questions**

- "Explain Rust's ownership model."
- "What does the borrow checker do?"
- "How does Rust prevent data races?"

---

## 11. Going deeper

- [*The Rust Programming Language*, chapters 4, 10 and 15](https://doc.rust-lang.org/book/)
- Jon Gjengset, *Rust for Rustaceans* — intermediate depth on ownership and lifetimes.
- [The Rustonomicon](https://doc.rust-lang.org/nomicon/) — for when you eventually need `unsafe`.

**Next:** [Chapter 3 — Types, Traits and Error Handling](03-types-traits-and-error-handling.md) covers how
Rust models data, behavior and failure.
