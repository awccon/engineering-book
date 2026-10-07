# How Programs Run: The .NET Runtime, IL and the JIT

> **🔄 Current (as of October 2026):** .NET 10 is the current Long-Term Support (LTS)
> release, shipped November 2025 with C# 14. .NET 11 is in preview and is expected in
> November 2026. Code in this chapter targets .NET 10 (`net10.0`).

You have probably written a lot of C# without ever needing to think about what happens
after you press **Run**. That is a sign the platform is doing its job. But sooner or
later that knowledge gap shows up as a real problem:

- An API is fast in every benchmark you run, but the first request after each deployment
  takes three seconds.
- A container image is 220 MB and takes 400 ms to start, and someone asks whether it
  could be 20 MB and start in 20 ms.
- The app runs on your laptop but fails on the server with *"You must install or update
  .NET to run this application."*
- An interviewer asks: *"What's the difference between the CLR, IL and the JIT?"*

This chapter builds the mental model underneath all of those. It is the foundation for
the rest of Book I: generics, async/await, the garbage collector and performance all make
much more sense once you know what the runtime is and what it does on your behalf.

---

## 1. The problem: why have a runtime at all?

Start with what a CPU actually executes: machine code for one specific instruction set
(x64, ARM64) running under one specific operating system (Windows, Linux, macOS). A
program compiled from C or C++ is turned directly into that machine code. It's fast and
has no middleman, but it brings three hard problems:

1. **Portability.** The binary works on exactly one CPU + OS combination. Ship to
   another and you must compile again.
2. **Memory safety.** Nothing stops code from reading past the end of an array, using
   memory after freeing it, or freeing it twice. These bugs are the source of a large
   share of serious security vulnerabilities in systems software.
3. **Productivity.** You manage memory by hand, and every library has its own conventions
   for strings, errors and threads.

A **managed runtime** is a deliberate trade: put a layer of software between your program
and the machine, and let that layer take on responsibilities you would otherwise carry
yourself. Java took this approach with the JVM in 1995. Microsoft took it with .NET in
2002. The layer costs something (startup time, memory, some loss of control), and in
exchange it gives you safety, portability and a large, consistent standard library.

> **🧱 Durable:** "What does the runtime do for me, and what does it cost?" is the single
> most useful question to ask about any platform: .NET, the JVM, Node.js, Python, Go,
> or Rust (which deliberately has almost no runtime, as Book XII explains).

---

## 2. The mental model: from source code to machine code

Here is the whole journey. Keep this picture in mind; every later section zooms in on one
step of it.

```text
 BUILD TIME (your machine / CI)                RUN TIME (wherever the app runs)
 ─────────────────────────────                 ─────────────────────────────────

  Program.cs  ─┐                                 ┌─────────────── .NET runtime (CLR) ───────────────┐
  Ticket.cs   ─┼─► Roslyn compiler ─► Beacon.dll │                                                  │
  ...         ─┘   (csc)              ┌────────┐ │  load assembly ─► verify ─► JIT ─► machine code  │
                                      │metadata│─┼─►    (loader)     (types)   (per    runs on CPU  │
                                      │   IL   │ │                            method)               │
                                      └────────┘ │  + garbage collector, exceptions, threads,       │
                                                 │    type system, interop, security checks         │
                                                 └──────────────────────────────────────────────────┘
```

Two compilers are involved, at two different times:

| Step | Tool | When | Input → Output |
|---|---|---|---|
| 1 | **Roslyn** (the C# compiler) | Build time | C# source → **IL + metadata**, packaged in an **assembly** (`.dll`) |
| 2 | **RyuJIT** (the JIT compiler) | Run time, usually | IL → **native machine code** for the exact CPU it's running on |

That two-step design is the key idea of .NET. The rest of this chapter explains each piece.

### Vocabulary you'll need

- **CLR (Common Language Runtime)** — the runtime engine: loader, JIT, garbage collector,
  exception handling, threading, type system. In modern .NET the implementation is called
  **CoreCLR**.
- **IL (Intermediate Language)**, also called **CIL** or **MSIL** — a CPU-independent
  instruction set that all .NET languages (C#, F#, VB) compile into.
- **Metadata** — tables inside the assembly that describe every type, method, field,
  parameter and reference. This is what makes reflection, IntelliSense and the debugger
  possible.
- **Assembly** — the unit of deployment and versioning: a `.dll` (or `.exe`) containing IL,
  metadata and a manifest (name, version, dependencies).
- **BCL (Base Class Library)** — `System.String`, `List<T>`, `HttpClient`, `File` and the
  rest of the standard library that ships with the runtime.
- **SDK** — the developer toolchain: the `dotnet` CLI, compilers, MSBuild, templates. You
  need it to *build*; servers only need the *runtime* to *run*.

---

## 3. Step one: Roslyn turns C# into IL

Consider the smallest interesting method:

```csharp
static int Add(int a, int b) => a + b;
```

Roslyn compiles it into this IL (Release build):

```text
.method private hidebysig static int32 Add(int32 a, int32 b) cil managed
{
  .maxstack 2
  IL_0000: ldarg.0      // push argument 0 (a) onto the evaluation stack
  IL_0001: ldarg.1      // push argument 1 (b)
  IL_0002: add          // pop two values, push their sum
  IL_0003: ret          // return the value on top of the stack
}
```

Three things to notice:

1. **IL is a stack machine.** Instructions push and pop values on an abstract
   *evaluation stack*. Real CPUs use registers, so IL can't run directly on hardware. It
   is a portable description of *what* to compute, not *how* a specific CPU should do it.
2. **IL is typed.** `int32` appears in the signature, and the runtime can verify that
   instructions are applied to values of the right types. That is the basis of .NET's
   *type safety*: you cannot treat an `int` as a pointer or a `string` as a `Ticket`.
3. **Roslyn does very little optimization.** It folds constants and removes dead code,
   but the heavy optimization (inlining, register allocation, loop optimization) is left
   to the JIT, which knows the actual CPU and can observe the program running.

### Debug vs Release is mostly about this step

In a **Debug** build, Roslyn emits extra `nop` instructions (places to put breakpoints),
keeps every local variable alive so you can inspect it, and tells the JIT to optimize
less. In a **Release** build, it doesn't. That's why:

> **⚠️ What can go wrong:** Measuring performance in a Debug build tells you almost nothing.
> Always benchmark Release builds, and preferably use BenchmarkDotNet (Chapter 11), which
> refuses to run benchmarks on Debug builds for exactly this reason.

### C# features are compiler tricks

A surprising amount of C# exists only at the Roslyn level. The runtime never sees it:

| You write | Roslyn emits |
|---|---|
| `record Ticket(...)` | An ordinary class with generated `Equals`, `GetHashCode`, `ToString`, `Deconstruct` and a copy method |
| `foreach` over a `List<T>` | A `while` loop over `GetEnumerator()` / `MoveNext()` / `Current` |
| `async` / `await` | A generated **state machine** struct (Chapter 9 takes this apart) |
| Lambdas capturing variables | A generated **closure** class holding the captured variables (Chapter 6) |
| `using var x = ...` | A `try` / `finally` that calls `Dispose()` |
| String interpolation `$"..."` | Calls to an interpolated-string handler that builds the string efficiently |

This is why decompilers such as ILSpy and the website **SharpLab** are such good learning
tools: they show you what your "nice" C# really turns into. We'll use them throughout
Book I.

---

## 4. Step two: the runtime loads and runs your assembly

When you run `dotnet Beacon.Cli.dll` (or the `Beacon.Cli` executable, which is a small
native *apphost* that does the same thing), this happens:

1. **The host starts.** `dotnet` (the *muxer*) reads `Beacon.Cli.runtimeconfig.json` to
   find out which runtime version the app needs, finds a matching installed runtime, and
   loads it.
2. **The CLR initializes.** It sets up the garbage-collected heap, the thread pool, and
   the type system.
3. **The assembly loader** opens `Beacon.Cli.dll`, reads its metadata, and resolves
   dependencies (`Beacon.Core.dll`, `System.Runtime.dll`, …) using
   `Beacon.Cli.deps.json`.
4. **The entry point is found** (`Main`, or your top-level statements, which Roslyn wraps
   in a generated `Main`).
5. **The JIT compiles `Main`** into machine code, and execution begins.
6. **Every method is compiled on first call.** Until a method is called, it's just IL.
   The first call goes through a small *stub* that invokes the JIT; the JIT compiles the
   method, patches the stub to point at the new machine code, and every later call goes
   straight to native code.

Point 6 is the most important and the most often misunderstood. **.NET code is not
interpreted.** Every method runs as real machine code. It is just compiled *lazily*, the
first time it's needed.

### What the CLR does while your code runs

Once running, the runtime keeps providing services that, in C or C++, would be your job:

- **Memory management.** You allocate with `new`; the **garbage collector (GC)** reclaims
  objects that are no longer reachable. No `free`, no use-after-free, no double free.
  (Chapter 11 explains how the GC decides, and what it costs.)
- **Type safety.** Casts are checked (`InvalidCastException`), arrays are bounds-checked
  (`IndexOutOfRangeException`), and null dereferences become `NullReferenceException`
  instead of corrupting memory.
- **Exception handling.** Unwinding the stack, running `finally` blocks, and finding the
  right `catch` across method boundaries.
- **Threading.** Managed threads, the thread pool, and the memory model that makes
  `lock`, `volatile` and `Interlocked` work.
- **Interop.** Calling native libraries (P/Invoke) and marshaling data across the boundary.

That is what **managed code** means: code whose execution is managed by the runtime.
**Unmanaged code** is everything outside it, such as a C library you call through P/Invoke
or the operating system itself. Most bugs that cross the boundary (leaked handles,
crashes in native code) are bugs about who owns what.

---

## 5. Inside the JIT: why the same code gets faster as it runs

A JIT has a tension built in. Compiling *fast* means the app starts quickly but runs
slower code. Compiling *well* produces fast code but delays startup, because heavy
optimization takes time. Modern .NET resolves this with **tiered compilation**:

```text
 first call                hot (called ~30+ times)           
 ──────────►  Tier 0  ───────────────────────────────►  Tier 1
              quick, barely                              fully optimized,
              optimized code                             guided by what the
              + instrumentation                          program actually did
```

- **Tier 0.** The first time a method runs, the JIT compiles it quickly with minimal
  optimization. Startup stays fast.
- **Counting.** The runtime counts calls. Methods called often enough (around 30 calls)
  are queued for recompilation on a background thread.
- **Tier 1.** The method is recompiled with full optimization, and calls are redirected
  to the new code. Your app never pauses for this.
- **Dynamic PGO (Profile-Guided Optimization).** Tier 0 code can carry instrumentation
  that records what actually happens: which branches are taken, which concrete type an
  interface call usually hits. Tier 1 then optimizes for that real behavior. For
  example, if an `IRepository` call almost always lands on `PostgresRepository`, the JIT
  can add a fast path that calls (and even inlines) that implementation directly, with a
  fallback for everything else. This is on by default since .NET 8.
- **OSR (On-Stack Replacement).** A method with a long-running loop, such as `Main` with a
  big processing loop, might be called only once and so never get "hot." OSR lets the
  runtime switch to optimized code *in the middle of the loop*.

> **🧱 Durable:** A JIT can do things an ahead-of-time compiler can't, because it sees the
> real machine (it can use AVX-512 if the CPU has it) and the real workload (PGO). An
> ahead-of-time compiler starts faster and more predictably, because no compilation
> happens at run time. Neither is "better"; they optimize for different things.

### What the JIT does to your code

Some of the most important optimizations, which you'll see referenced throughout the book:

- **Inlining** — replacing a call to a small method with the method's body. This removes
  call overhead and, more importantly, unlocks further optimizations. It's why small
  properties and helper methods are essentially free.
- **Devirtualization** — turning an interface or virtual call into a direct call when the
  JIT can prove (or, with PGO, guess and check) the concrete type. `sealed` classes help.
- **Bounds-check elimination** — removing the array bounds check when the JIT can prove
  the index is in range, as in `for (int i = 0; i < array.Length; i++)`.
- **Generic specialization** — generating separate machine code for each value type a
  generic is used with (`List<int>`, `List<double>`) so there is no boxing. Chapter 4
  explains this in depth.
- **Register allocation and vectorization** — keeping values in CPU registers and using
  SIMD instructions where possible.

### Seeing it for yourself

You can ask the runtime to print the machine code the JIT generates. Set an environment
variable before running a Release build:

```bash
# Linux/macOS
DOTNET_JitDisasm="Add" dotnet run -c Release

# PowerShell
$env:DOTNET_JitDisasm="Add"; dotnet run -c Release
```

You'll see output for each tier. For `Add` on x64, the fully optimized version is
essentially a single instruction plus a return, something like:

```text
; Tier1 code
       lea      eax, [rdi+rsi]
       ret
```

(Exact registers vary by OS and CPU. On Windows, arguments arrive in `ecx` and `edx`.)
Four IL instructions became two machine instructions. If `Add` were called from another
method, the JIT would most likely inline it, and the call would disappear entirely.

---

## 6. Ahead-of-time options: ReadyToRun and Native AOT

JIT compilation is a run-time cost. Sometimes you'd rather pay it at build time. .NET
gives you two main options.

### ReadyToRun (R2R)

```xml
<PublishReadyToRun>true</PublishReadyToRun>
```

The publish step precompiles IL into native code and stores it *alongside* the IL in the
assembly. At startup the runtime uses the precompiled code, so less JIT work happens.
The IL is still there, so tiered compilation can still recompile hot methods into better
Tier 1 code later. The .NET libraries themselves ship as ReadyToRun, which is a big part
of why startup is reasonable even for large apps.

**Trade-off:** larger files, and the precompiled code is less optimized than Tier 1 (it
targets a generic CPU and has no PGO data). It's a startup optimization, not a throughput
optimization.

### Native AOT

```xml
<PublishAot>true</PublishAot>
```

The publish step compiles the *entire application*, including the parts of the runtime
and libraries it uses, into a single native executable. There is no JIT and no IL at run
time. The result:

- **Very fast startup** (often tens of milliseconds instead of hundreds),
- **lower memory use**,
- **a single, self-contained file** with no .NET installation needed on the target,
- usually a **smaller deployment** than a self-contained JIT app, because unused code is
  *trimmed* away.

But you pay for it:

- **No runtime code generation.** Anything that generates code at run time
  (`Reflection.Emit`, some dynamic proxies, runtime expression compilation) won't work
  or falls back to slower interpretation.
- **Reflection is restricted.** The trimmer removes code it can't see being used. If a
  library discovers types by reflection, as many older serializers, DI containers and
  ORMs do, those types may be trimmed away. The build emits **trim and AOT warnings** for
  this; treat them as errors.
- **No dynamic PGO**, so steady-state throughput can be somewhat lower than a warmed-up
  JIT app.
- **You build for one platform** (for example `linux-x64`); there's no portable IL.

> **🔄 Current (as of October 2026):** ASP.NET Core supports Native AOT for minimal APIs,
> gRPC and worker services, with some features unsupported (notably MVC controllers and
> Razor). `System.Text.Json` supports AOT through **source generators**, which do at
> compile time what reflection used to do at run time. EF Core's AOT support is still
> limited. Check the current compatibility list before choosing AOT for a web app.

### Choosing

| Situation | Usually the best fit |
|---|---|
| A typical web API or long-running service | **Default JIT**. Startup happens once; steady-state throughput and full library compatibility matter more. |
| Big app where startup matters (desktop, CLI with many dependencies) | **ReadyToRun** |
| Serverless functions, scale-to-zero containers, small CLI tools, sidecars | **Native AOT**, if your dependencies are compatible |
| Plugin systems, heavy reflection, runtime code generation | **JIT only** |

> **🧭 When not to use it:** Don't adopt Native AOT because it sounds faster. For a web
> API that runs for days, startup time is irrelevant and a warmed-up JIT with PGO is
> often *faster*. AOT pays off when you start processes often (serverless, autoscaling to
> zero, command-line tools) or when memory and image size directly cost money.

---

## 7. SDK, runtime and the way apps are deployed

Many confusing deployment errors come from mixing up these pieces.

```text
  .NET SDK  (developer machines, CI)
  ├── dotnet CLI, MSBuild, Roslyn, NuGet, templates
  └── includes a runtime, so you can run what you build

  .NET Runtime  (servers)
  ├── Microsoft.NETCore.App        ← the base runtime + BCL (console apps, workers)
  └── Microsoft.AspNetCore.App     ← ASP.NET Core, on top of the base runtime
```

### Target framework

Your project file declares which framework it targets:

```xml
<TargetFramework>net10.0</TargetFramework>
```

That **TFM (target framework moniker)** decides which APIs you can use at compile time
and which runtime version the app requires at run time. By default an app built for
`net10.0` runs on the latest installed **10.0.x patch** of the runtime (so security
patches apply without rebuilding), but it will *not* run on .NET 9.

### Framework-dependent vs self-contained

| | Framework-dependent (default) | Self-contained | Native AOT |
|---|---|---|---|
| Needs .NET installed on the server? | Yes | No | No |
| Size | Small (just your code) | Large (your code + runtime) | Medium, single file |
| Who patches the runtime? | Whoever maintains the server | **You**, by republishing | **You**, by republishing |
| Platform-specific? | No (IL is portable) | Yes | Yes |

> **⚠️ What can go wrong:** *"You must install or update .NET to run this application"*
> means a framework-dependent app found no compatible runtime. The usual causes are a
> server with the wrong major version, or one with only `Microsoft.NETCore.App` installed
> when the app needs `Microsoft.AspNetCore.App`. Run `dotnet --list-runtimes` on the
> server, compare with the app's `.runtimeconfig.json`, and either install the right
> runtime or publish self-contained. In containers this problem largely disappears,
> because the base image pins the runtime (Book VIII).

> **⚠️ What can go wrong:** Self-contained apps don't get runtime security patches from
> the OS. If you publish self-contained, your CI pipeline must rebuild and redeploy when
> .NET ships a security update. This is a common gap in real teams.

### Release cadence

.NET ships a new major version every November. **Even-numbered** versions are
**LTS (Long-Term Support)**; **odd-numbered** versions are **STS (Standard-Term Support)**
with a shorter support window. Monthly patch releases ("Patch Tuesday") carry security
fixes. Production systems usually target the current LTS and plan an upgrade every two
years.

> **🔄 Current (as of October 2026):** see the
> [official .NET support policy](https://dotnet.microsoft.com/platform/support/policy/dotnet-core)
> for exact end-of-support dates.

---

## 8. In practice: starting Beacon

Time to create the first piece of the running project. Beacon begins as two projects:
a class library for the domain (`Beacon.Core`) and a console app that uses it
(`Beacon.Cli`). Later books add the API, the database and the frontend around this core.

```bash
mkdir beacon && cd beacon
dotnet new sln -n Beacon
dotnet new classlib -n Beacon.Core -o src/Beacon.Core
dotnet new console  -n Beacon.Cli  -o src/Beacon.Cli
dotnet sln add src/Beacon.Core src/Beacon.Cli
dotnet add src/Beacon.Cli reference src/Beacon.Core
```

Delete the template's `Class1.cs` and add the first domain type:

```csharp
// src/Beacon.Core/Tickets/Ticket.cs
namespace Beacon.Core.Tickets;

public enum TicketStatus { Open, InProgress, Resolved, Closed }

public sealed record Ticket(int Id, string Title, TicketStatus Status)
{
    public bool IsActive => Status is TicketStatus.Open or TicketStatus.InProgress;
}
```

```csharp
// src/Beacon.Cli/Program.cs
using Beacon.Core.Tickets;

Ticket[] tickets =
[
    new(1, "Cannot log in", TicketStatus.Open),
    new(2, "Export to PDF is slow", TicketStatus.InProgress),
    new(3, "Typo on the pricing page", TicketStatus.Resolved),
];

foreach (var ticket in tickets)
{
    Console.WriteLine($"#{ticket.Id,-3} {ticket.Status,-10} {ticket.Title}");
}

Console.WriteLine($"Active tickets: {tickets.Count(t => t.IsActive)}");
```

Run it:

```bash
dotnet run --project src/Beacon.Cli
```

```text
#1   Open       Cannot log in
#2   InProgress Export to PDF is slow
#3   Resolved   Typo on the pricing page
Active tickets: 2
```

### Look under the hood

Now use what this chapter covered. Build in Release and look at what was produced:

```bash
dotnet build -c Release
ls src/Beacon.Cli/bin/Release/net10.0/
```

You'll find:

- `Beacon.Cli.dll` and `Beacon.Core.dll` — your IL and metadata.
- `Beacon.Cli` (or `Beacon.Cli.exe` on Windows) — the small native apphost.
- `Beacon.Cli.runtimeconfig.json` — which runtime the app needs.
- `Beacon.Cli.deps.json` — the dependency graph the loader uses.

Open `runtimeconfig.json`:

```json
{
  "runtimeOptions": {
    "tfm": "net10.0",
    "framework": {
      "name": "Microsoft.NETCore.App",
      "version": "10.0.0"
    }
  }
}
```

That is the file step 1 of section 4 reads at startup.

Next, see what Roslyn did with the `record`. Install the ILSpy command-line tool and
decompile `Beacon.Core.dll`:

```bash
dotnet tool install -g ilspycmd
ilspycmd src/Beacon.Core/bin/Release/net10.0/Beacon.Core.dll
```

You'll see that the one-line `record` became a full class with a constructor,
properties, `Equals(Ticket?)`, `GetHashCode()`, `ToString()`, `PrintMembers`,
`Deconstruct`, the `==` and `!=` operators and a copy method used by `with` expressions.
Add `-il` to see the raw IL instead. Alternatively, paste the record into
[SharpLab](https://sharplab.io) and switch the output between *C#*, *IL* and *JIT Asm*.

Finally, try the publishing options and compare:

```bash
# Framework-dependent (default)
dotnet publish src/Beacon.Cli -c Release -o out/fdd

# Self-contained, single file, for Linux x64
dotnet publish src/Beacon.Cli -c Release -r linux-x64 --self-contained \
  -p:PublishSingleFile=true -o out/self

# Native AOT (needs the platform's native toolchain: clang on Linux, VS C++ tools on Windows)
dotnet publish src/Beacon.Cli -c Release -r linux-x64 -p:PublishAot=true -o out/aot
```

Compare the sizes with `du -sh out/*` and the startup times with `time ./out/aot/Beacon.Cli`
versus `time dotnet out/fdd/Beacon.Cli.dll`. Write the numbers down; you'll have
real evidence for the trade-off table in section 6 instead of a memorized claim.

---

## 9. What can go wrong

A summary of the runtime-level problems you will actually meet:

- **Cold start latency.** The first request to a freshly deployed API is slow because
  hundreds of methods are being JIT-compiled at Tier 0, assemblies are being loaded and
  caches are empty. *Fixes:* warm-up requests in your deployment (health checks that hit
  real code paths), ReadyToRun, or Native AOT where startup truly matters.
- **Benchmarks that lie.** Debug builds, a single timing run (which measures the JIT, not
  your code), or code the JIT optimized away because its result was never used. *Fix:*
  BenchmarkDotNet, always in Release.
- **Runtime version mismatches** on servers, covered in section 7.
- **Trimming and AOT surprises.** Code that works under `dotnet run` fails after
  publishing with trimming because a type used only through reflection was removed. *Fix:*
  treat trim/AOT warnings as errors; prefer libraries with source generators.
- **Assuming "managed" means "no leaks."** The GC frees *memory*, not *resources*. File
  handles, sockets and database connections still need `Dispose()`. And an object that is
  still referenced (from a static list, an event subscription, a cache) is never
  collected. Chapter 11 covers both.
- **Native interop crashes.** A bug in native code called through P/Invoke can take down
  the whole process, and the runtime's safety guarantees don't apply on the other side of
  that boundary.

---

## 10. How an experienced engineer thinks about this

You will rarely think about IL or the JIT while writing a feature, and that's correct.
The knowledge pays off in specific moments:

- **Performance questions start with measurement and an accurate model.** "Is this
  interface call slow?" Often not: the JIT may devirtualize and inline it. "Is LINQ slow?"
  It allocates enumerators and delegates; whether that matters depends on how hot the
  path is. The model tells you what *might* be expensive; the profiler tells you what
  *is*.
- **Deployment choices are trade-offs, not upgrades.** JIT, ReadyToRun and Native AOT
  optimize for different things. Experienced engineers ask how often the process starts,
  how long it lives, how much memory costs, and which libraries it depends on, and only
  then choose.
- **Know which layer a feature lives in.** Records, `async`, lambdas and pattern matching
  are compiler features; the GC, type safety and the JIT are runtime features. When
  something behaves surprisingly, knowing the layer tells you where to look: decompile
  for compiler questions, profile or read runtime docs for runtime questions.
- **Patching is part of the design.** Choosing self-contained or AOT also means choosing
  to own runtime security updates.

---

## 11. Check yourself

**Questions**

1. What two compilers turn C# into running machine code, and when does each run?
2. What is in a .NET assembly besides IL? Name one feature that depends on it.
3. Is .NET code interpreted? Explain what happens on the first and the hundredth call of
   a method.
4. What problem does tiered compilation solve? What does Dynamic PGO add?
5. Why can a JIT-compiled app sometimes run *faster* than an ahead-of-time compiled one?
6. Give two concrete reasons Native AOT might break an app that works fine with the JIT.
7. A colleague publishes a self-contained app to save the operations team from
   installing .NET. What new responsibility has the team taken on?
8. Name three C# features that exist only at the compiler level.

**Exercises**

1. In SharpLab, write a `foreach` over a `List<int>` and over an `int[]`. Compare the
   lowered C#. Why are they different?
2. Write a lambda that captures a local variable, decompile it, and find the generated
   closure class.
3. Run the three `dotnet publish` variants from section 8 and record size and startup time
   for each in a small table.
4. Use `DOTNET_JitDisasm` to look at a method containing a `for` loop over an array.
   Can you find the bounds check? Now change the loop condition to a hard-coded number
   larger than the array length and look again.

**Interview-style questions**

- "Explain the difference between the CLR, the BCL, IL and the JIT."
- "What's the difference between managed and unmanaged code?"
- "When would you choose Native AOT for a .NET service, and when wouldn't you?"
- "Why is the first request to our API slow after each deployment?"

---

## 12. Going deeper

- [.NET documentation: Managed execution process](https://learn.microsoft.com/dotnet/standard/managed-execution-process)
- [.NET documentation: Native AOT deployment](https://learn.microsoft.com/dotnet/core/deploying/native-aot/)
- [The Book of the Runtime](https://github.com/dotnet/runtime/tree/main/docs/design/coreclr/botr) — the
  CLR team's own design documentation. Dense, but authoritative.
- Stephen Toub's annual *Performance Improvements in .NET* posts on the .NET blog —
  the best way to stay current with what the JIT and runtime learn each year.
- [SharpLab](https://sharplab.io) — see C#, IL and machine code side by side in the browser.

**Next:** [Chapter 2 — Types, Values and References](02-types-values-and-references.md)
takes the type system the runtime enforces and looks at how values and objects actually
live in memory.
