# Maintainability, Technical Debt and Legacy Modernization

Successful software lives for a long time. Beacon's code will still be running, and still be changed,
long after the people who wrote the first version have moved on. Most professional engineering is not
greenfield work; it's changing systems that already exist, often ones built under different constraints,
by different people, with older tools.

This chapter is about keeping systems changeable: what maintainability actually is and how to measure
it, how to think about technical debt as a business decision rather than a moral failing, how to change
code you don't understand safely, and how to modernize legacy systems, including the very common case of
.NET Framework applications, without a big-bang rewrite.

---

## 1. The problem: software gets harder to change

Every team has seen it: in the first months a feature takes days; two years later a similar feature takes
weeks. Nobody decided to slow down. It happened through hundreds of small decisions: a shortcut under
deadline pressure, a dependency not upgraded, a design that fit last year's requirements, a module only
one person understood, tests that became too slow to run.

Lehman's laws of software evolution (1970s–80s) describe this precisely: a system that is used must be
continually adapted or it becomes progressively less useful, and as it evolves its complexity increases
**unless work is done to maintain or reduce it**. Entropy is the default. Maintainability is the result
of continuous, deliberate effort.

The opposite reaction is just as costly: "this code is terrible, let's rewrite it". Rewrites take longer
than planned, must reproduce years of undocumented behavior and edge cases, and deliver nothing new until
they're finished, while the old system still needs maintenance. Many rewrites are abandoned or end up
with a new system as complicated as the old one.

The skill is in between: **keep the system changeable through continuous small improvements, and modernize
incrementally when larger change is needed.**

---

## 2. The mental model: maintainability is the cost of the next change

Maintainability isn't elegance or adherence to patterns. It's **how expensive and risky the next change
is**. Several properties contribute:

- **Understandability**: can a developer find where to make the change and predict its effects?
  (Names, structure, consistency, documentation of the non-obvious.)
- **Modularity**: is the change local? (Chapter 1's coupling and cohesion, Chapter 2's boundaries.)
- **Testability**: can you verify the change quickly and confidently?
- **Deployability**: can you ship it safely, frequently, and roll it back?
- **Currency**: are the platform, frameworks and dependencies supported and patched?
- **Knowledge distribution**: does more than one person understand each part?

> **🧱 Durable:** The best single indicator of maintainability is **how long it takes, and how risky it
> feels, to make and ship a small change**. A team that can safely deploy a one-line fix to production in
> under an hour, with confidence, has a maintainable system, whatever the code looks like. A team that
> needs a two-week release train and a manual test phase does not, however clean the code.

### Measuring it

You can't manage what you can't see. Useful measurements, from most to least reliable:

**Delivery metrics (DORA).** The DevOps Research and Assessment program found four metrics that predict
both delivery performance and organizational outcomes: **deployment frequency**, **lead time for
changes** (commit to production), **change failure rate** and **time to restore service**. (Later reports
added reliability and rework rate.) They measure the whole system of code, tests, pipeline and process.
If lead time grows and change failure rate rises, maintainability is falling, wherever the cause is.

**Hotspots: churn × complexity.** Adam Tornhill's insight: complexity only matters where code changes.
A 3,000-line class nobody touches costs nothing. A 3,000-line class modified in 40% of pull requests is
where your debt interest is paid. Git history tells you where:

```bash
# Files changed most often in the last year (churn)
git log --since="1 year ago" --name-only --pretty=format: -- '*.cs' '*.ts' '*.tsx' \
  | grep -v '^$' | sort | uniq -c | sort -rn | head -20
```

Combine churn with a complexity measure (lines of code, cyclomatic complexity, indentation depth) and
plot them: the top-right corner (high churn, high complexity) is where refactoring pays off. Tools like
CodeScene automate this, and also find **change coupling** (files that always change together across
module boundaries, a sign of a misplaced boundary) and **knowledge islands** (code only one person has
touched).

**Code-level metrics** (coverage, complexity, duplication, static analysis warnings) are useful as
trends and for spotting outliers, but poor as targets. Goodhart's law applies: a coverage target of 80%
produces tests that execute code without asserting anything.

**Developer experience.** Ask the team regularly: which parts of the system do you dread changing? The
answers usually match the hotspot analysis, and add context the data can't.

---

## 3. Technical debt

Ward Cunningham coined the metaphor in 1992: shipping code that reflects an incomplete understanding is
like taking on debt. It lets you move faster now; you pay **interest** (extra effort on every later change
in that area) until you pay back the **principal** (refactor the code to reflect what you've since
learned).

The metaphor's value is that it frames code quality as a **financial decision**, which makes it
discussable with non-engineers. Debt isn't inherently bad. Taking on debt to hit a market window can be
the right call, if you know you're doing it and plan to pay it back.

### Kinds of debt

Martin Fowler's **technical debt quadrant** separates how debt arises:

|  | **Reckless** | **Prudent** |
|---|---|---|
| **Deliberate** | "We don't have time for design." | "We must ship now and deal with the consequences." |
| **Inadvertent** | "What's layering?" | "Now we know how we should have done it." |

Prudent-inadvertent debt is unavoidable: you learn as you build. Prudent-deliberate debt is a legitimate
business choice. Reckless debt is the expensive kind, and it's a skills or culture problem more than a
code problem.

Debt also comes in different forms, beyond messy code:

- **Design debt**: boundaries in the wrong place, a model that no longer fits the domain.
- **Test debt**: missing or slow tests, so every change is risky.
- **Dependency and platform debt**: outdated frameworks, end-of-support runtimes, vulnerable packages.
  This kind accrues interest even if you change nothing, because the world moves.
- **Infrastructure debt**: manual deployments, snowflake servers, missing monitoring.
- **Documentation and knowledge debt**: decisions nobody recorded, systems only one person understands.
- **Data debt**: inconsistent data, missing constraints, columns that mean different things for
  different rows.

### Managing debt

**Make it visible.** Keep a lightweight debt register: what, where, its **interest** (how it slows the
team or creates risk, with evidence from hotspots, incidents or lead time), and a rough cost to fix. Link
debt items to the incidents and delays they cause. "This module caused 4 of last quarter's 9 incidents"
is a business argument; "this code is ugly" isn't.

**Prioritize by interest, not by ugliness.** Fix debt in hotspots and on the path of upcoming features.
Leave ugly-but-stable code alone.

**Pay continuously.** Patterns that work:

- **The boy scout rule**: leave code a little better than you found it, in the area you're already
  changing. Rename the confusing variable, extract the method, add the missing test. Small and
  continuous beats large and occasional.
- **Refactor before the feature** ("make the change easy, then make the easy change", Kent Beck): when a
  feature lands in a messy area, first refactor (separate PR, no behavior change), then add the feature.
- **A standing allocation**: many teams reserve a fixed share of capacity (often 15–25%) for debt,
  upgrades and tooling, so it doesn't compete with features every sprint.
- **Debt as part of the feature estimate**: if building the feature well requires restructuring, that's
  part of the feature's cost, not a separate "nice to have".

**Stay current.** Platform and dependency debt is the easiest to prevent: automated dependency updates
(Renovate, Dependabot), a habit of upgrading to each .NET LTS release within months of its release, and
CI that runs against the next version early.

> **🔄 Current (as of October 2026):** .NET 10 (LTS, released November 2025) is supported until November
> 2028. .NET 8 (LTS) and .NET 9 (STS, whose support Microsoft extended to 24 months) both reach end of
> support on **November 10, 2026**, so applications still on them should be upgrading now. .NET 11 is
> expected in November 2026 as an STS release. .NET Framework 4.8.1 remains supported as a Windows
> component, but receives no new features. Check the [.NET support policy](https://dotnet.microsoft.com/platform/support/policy)
> for current dates.

> **⚠️ What can go wrong:** "Refactoring sprints" and "tech debt quarters" tend to fail: they're too long
> to stay focused, too detached from business needs to keep support, and the debt returns because the
> habits that created it haven't changed. Big cleanups occasionally make sense (a framework migration),
> but run them like features: a goal, measurable outcomes, incremental delivery.

---

## 4. Changing code you don't understand

Michael Feathers defined legacy code as **code without tests**. Without tests, every change is a guess,
so people make the smallest possible edit in the most local place, and the structure gets worse. The way
out is to get the code under test, then refactor safely.

### Characterization tests

A **characterization test** records what the code **currently does**, not what it should do. You're not
judging correctness; you're building a safety net so that refactoring doesn't change behavior
unintentionally.

1. Call the code with an input.
2. Write an assertion you expect to fail.
3. Let the failure tell you the actual output.
4. Change the assertion to match. Now the behavior is pinned.

**Snapshot (approval) testing** speeds this up for code with complex outputs: Verify (for .NET) serializes
the result to a file on the first run; you review and approve it; later runs fail on any difference.

```csharp
public sealed class LegacySlaCalculatorTests
{
    [Theory]
    [MemberData(nameof(RealisticCases))]          // sampled from production data, anonymized
    public Task Calculates_the_same_due_dates_as_before(LegacyTicketInput input) =>
        Verify(new LegacySlaCalculator().Calculate(input))
            .UseParameters(input.CaseId);         // one approved snapshot per case
}
```

A good source of inputs is **production data**: sample real requests or records (anonymized), run them
through the code, and approve the outputs. They capture edge cases nobody would think to write.

> **⚠️ What can go wrong:** Characterization tests also pin **bugs**. When one fails after a refactoring,
> check whether you changed behavior accidentally or fixed a bug intentionally. Fix bugs in separate,
> clearly labeled commits after the refactoring, so the behavior change is visible in review and can be
> reverted on its own.

### Seams

A **seam** (Feathers) is a place where you can change behavior without editing the code there: a
constructor parameter, an interface, a virtual method, a configuration value. Legacy code often has
none: it news up its dependencies, calls `DateTime.Now`, reads `HttpContext.Current`, and opens database
connections in the middle of business logic.

Introduce seams with small, safe, mechanical refactorings, ideally using the IDE's automated refactorings,
which are much less likely to introduce mistakes than hand edits:

- **Extract method** to isolate the logic you need to test.
- **Parameterize constructor**: pass in the dependency instead of creating it, with a default that
  preserves old behavior for existing callers.
- **Extract interface** around a static or concrete dependency (`IClock` around `DateTime.Now`, or use
  `TimeProvider`).
- **Sprout method / sprout class**: write new logic in a new, tested method or class, and call it from the
  legacy code with a one-line change.

```csharp
// Legacy: untestable because it reads the clock and the database inline
public decimal CalculatePenalty(int ticketId)
{
    var ticket = new TicketDal().Load(ticketId);
    var hoursLate = (DateTime.Now - ticket.DueAt).TotalHours;
    // ... 200 lines of rules ...
}

// Step 1: sprout the rules into a pure, tested method; the legacy method becomes a thin shell
public decimal CalculatePenalty(int ticketId)
{
    var ticket = new TicketDal().Load(ticketId);
    return PenaltyRules.Calculate(ticket.ToPenaltyInput(), DateTime.Now);
}

public static class PenaltyRules
{
    public static decimal Calculate(PenaltyInput input, DateTime now) { /* the 200 lines, now testable */ }
}
```

This is Chapter 1's "functional core, imperative shell", applied in reverse to existing code.

### Refactoring safely

- **Small steps, each one compiling and passing tests.** Commit often. If a step goes wrong, revert it
  rather than debugging it.
- **Separate refactoring from behavior change**, in separate commits or PRs (Book II, Chapter 3).
- **Use automated refactorings** (rename, extract, move, inline) wherever possible.
- **Work in the hotspots**, where the payoff is.
- **Use AI assistants carefully** (Book XI, Chapter 11): they're good at explaining unfamiliar code,
  drafting characterization tests and doing mechanical transformations at scale, but every change still
  needs the safety net of tests and a human review. Large AI-generated refactorings without tests are a
  fast way to introduce subtle behavior changes.

---

## 5. Legacy modernization: the strangler fig

When a system needs more than incremental refactoring (an unsupported platform, an architecture that
can't meet new requirements, a technology nobody can hire for), the temptation is a rewrite. The safer
approach is Martin Fowler's **strangler fig**, named after a vine that grows around a tree and gradually
replaces it:

1. **Put a facade in front of the legacy system** that routes all traffic. At first, everything goes to
   the old system.
2. **Build new functionality, or re-implement one existing slice, in the new system.**
3. **Route that slice to the new system** through the facade.
4. **Repeat**, slice by slice, until the old system handles nothing and can be switched off.

```text
Phase 1                    Phase 2                         Phase 3
Clients                    Clients                         Clients
   │                          │                               │
 Facade ───► Legacy        Facade ──┬──► Legacy (less)      Facade ───► New
                                    └──► New (more)          (legacy retired)
```

Why it works:

- **Value is delivered continuously**: each migrated slice is in production, used and tested by real
  traffic.
- **Risk is small per step**, and each step can be rolled back by changing a route.
- **You can stop** when the remaining legacy part is no longer worth migrating.
- **You learn** what the old system really does from production behavior, slice by slice.

### Supporting techniques

- **Branch by abstraction**: for replacing a component **inside** a codebase. Introduce an abstraction
  over the old implementation, switch callers to the abstraction, build the new implementation behind it,
  switch over (with a flag), then delete the old one. Everything happens on `main`, with no long-lived
  branch.
- **Parallel run (shadowing)**: send requests to both old and new implementations, return the old
  result, and compare. Differences reveal behavior you didn't know about. GitHub's Scientist library
  popularized this; it's easy to implement for read paths.
- **Dark launch**: deploy the new path to production behind a flag, exercised by internal users or a small
  percentage of traffic first.
- **Data synchronization**: during migration, old and new may both need data. Prefer one **owner** per
  data set at any time, with change data capture or events synchronizing a read copy to the other side.
  Two-way synchronization is a source of subtle bugs; avoid it if you can.
- **Anti-corruption layer** (Chapter 2): translate the legacy model at the boundary so its concepts don't
  leak into the new system.

> **🧭 When not to use it:** A full rewrite can be the right choice when the system is **small** (weeks of
> work, not years), its behavior is **well understood and well specified**, or the old system genuinely
> can't be routed around (an embedded system, a desktop application with no seams). Even then, run old and
> new in parallel and compare outputs before switching.

---

## 6. Modernizing .NET Framework applications

A large share of the .NET code running in companies today is still on .NET Framework 4.x: ASP.NET MVC 5
and Web API 2, Web Forms, WCF services, Windows services, EF6, often on Windows Server and IIS. Moving
to modern .NET brings large performance gains, Linux containers, current language features, and an
actively developed platform. Here's how experienced teams approach it.

### Step 1: Assess

- **Inventory projects and their types.** Class libraries and console apps usually port easily. ASP.NET
  MVC and Web API port with work. **Web Forms, WCF server and Workflow Foundation have no direct
  equivalent** in modern .NET and need re-architecture for those parts.
- **Find blockers**: Windows-only APIs (registry, WMI, `System.Drawing` on Linux), `System.Web`
  dependencies (`HttpContext.Current`) in business logic, AppDomains, .NET Remoting, third-party
  libraries with no modern .NET version, COM interop.
- **Map dependencies** between projects to find a migration order: leaf libraries first.

> **🔄 Current (as of October 2026):** Microsoft's modernization tooling has shifted toward AI-assisted
> upgrades in Visual Studio and GitHub Copilot ("app modernization" agents that analyze a solution, plan an
> upgrade, update project files and code, fix build errors and run tests), largely superseding the older
> .NET Upgrade Assistant. The tooling helps most with mechanical steps (SDK-style project conversion,
> package updates, API replacements); architecture decisions and verification remain your job. Check
> Microsoft's current "upgrade to .NET" documentation for the recommended tools.

### Step 2: Prepare in place

Many steps can be done **while still on .NET Framework**, each one shippable:

- Convert projects to **SDK-style** `.csproj` files.
- Move from `packages.config` to `PackageReference`.
- Update NuGet packages to versions that support both .NET Framework and modern .NET.
- Retarget shared libraries to **.NET Standard 2.0** (usable from both) or multi-target
  (`<TargetFrameworks>net48;net10.0</TargetFrameworks>`).
- Remove `HttpContext.Current` from business logic by passing what's needed explicitly (introducing
  seams).
- Add characterization tests around critical behavior.

### Step 3: Migrate incrementally

For web applications, Microsoft supports a strangler fig approach directly:

- Create a new **ASP.NET Core** application that becomes the front door, with **YARP** forwarding every
  route it doesn't handle yet to the existing ASP.NET application.
- Use the **System.Web adapters** (`Microsoft.AspNetCore.SystemWebAdapters`) so that migrated code can
  keep using familiar `System.Web` APIs during the transition, and so that both applications share
  **session state** and **authentication**, letting users move between old and new pages without noticing.
- Migrate controllers and pages one route at a time, starting with high-value or low-risk ones.

```csharp
// New ASP.NET Core front door (Program.cs, simplified)
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddSystemWebAdapters()
    .AddJsonSessionSerializer(o => o.RegisterKey<string>("CustomerName"))
    .AddRemoteAppClient(o =>
    {
        o.RemoteAppUrl = new(builder.Configuration["LegacyApp:Url"]!);
        o.ApiKey = builder.Configuration["LegacyApp:ApiKey"]!;
    })
    .AddSessionClient()
    .AddAuthenticationClient(isDefaultScheme: true);   // legacy app remains the authority for now

builder.Services.AddHttpForwarder();
builder.Services.AddControllersWithViews();

var app = builder.Build();
app.UseSystemWebAdapters();
app.MapControllers();                                    // migrated routes
app.MapForwarder("/{**catch-all}", app.Configuration["LegacyApp:Url"]!)
   .Add(static b => ((RouteEndpointBuilder)b).Order = int.MaxValue);   // everything else → legacy
app.Run();
```

For other application types:

| Legacy technology | Modern path |
|---|---|
| ASP.NET MVC 5 / Web API 2 | ASP.NET Core MVC or minimal APIs; mostly mechanical, with the incremental approach above |
| Web Forms | No port. Rebuild pages incrementally (Razor Pages, Blazor, or a React frontend) behind the same facade |
| WCF services | **CoreWCF** (community-supported port of WCF server) for compatibility, or move to gRPC / REST for new clients |
| WCF clients | `System.ServiceModel.*` client packages work on modern .NET |
| Windows services | Worker Service (`BackgroundService`) hosted as a Windows service, systemd unit or container |
| EF6 | EF6 runs on modern .NET (6.4+), so port the app first, then migrate to EF Core separately |
| ASP.NET Identity / Membership | ASP.NET Core Identity, or better, an external identity provider (Book III) |
| Configuration (`web.config`, `ConfigurationManager`) | `IConfiguration` and options; `ConfigurationManager` package as a bridge |

> **⚠️ What can go wrong:** Changing several things at once: runtime, framework, ORM, hosting (IIS to
> containers), operating system and architecture in one project. Each is a source of behavior changes;
> combined, failures are impossible to attribute. Change **one major thing at a time**, ship it, and
> stabilize before the next.

### Step 4: Decommission

The last 10% of a migration is often the hardest: rarely used features, reports nobody can explain,
integrations with forgotten systems. Use access logs to find what's actually used, talk to the business
about what can be retired rather than migrated, and **actually switch the old system off**. A migration
that leaves both systems running indefinitely has doubled your maintenance burden.

---

## 7. In practice: absorbing "Helpdesk Classic" into Beacon

Beacon's company acquires a competitor. Their product, "Helpdesk Classic", has 300 customers and runs on
ASP.NET MVC 5 and Web API 2 on .NET Framework 4.8, with a WCF service used by customers' integrations,
SQL Server with 400 stored procedures, and a Windows service that sends emails. The goal: move its
customers onto Beacon within 18 months without disrupting them.

**Assessment findings** (two weeks):

- Access logs show 60% of routes received **no traffic** in 90 days.
- The WCF service has 14 operations; customers actually call 5.
- The SLA calculation lives in a 3,000-line stored procedure and differs from Beacon's in subtle ways
  that customers depend on (business-hours calendars per customer site).
- Two engineers from the acquired team know the system; one is leaving in four months (a knowledge
  island and an urgent risk).

**Plan, as a strangler fig:**

1. **Knowledge first**: pair Beacon engineers with the departing engineer; record ADRs and a C4 diagram of
   Helpdesk Classic; capture characterization tests for the SLA procedure using a month of anonymized
   production tickets (snapshot tests via Verify).
2. **Facade**: put Front Door and an ASP.NET Core gateway with YARP in front of Helpdesk Classic.
   Nothing changes for customers, but routing is now under Beacon's control, and traffic is observable
   with OpenTelemetry.
3. **Integration API**: implement the 5 used WCF operations as a compatibility layer in Beacon (CoreWCF
   for unchanged SOAP contracts, so customers' integrations keep working), backed by Beacon's modules
   through an anti-corruption layer. Announce a REST alternative and a deprecation timeline.
4. **SLA rules**: extend Beacon's `SlaRules` with per-site business-hours calendars (a strategy, Chapter
   1). The characterization snapshots from step 1 run against Beacon's implementation in a **parallel
   run** for two months; 23 differences are found, of which 19 are fixed to match and 4 are confirmed
   with customers as legacy bugs.
5. **Tenant migration**: migrate customers in waves (internal test accounts, then 10 friendly customers,
   then cohorts of 50). Each tenant's data is copied into Beacon by a migration tool (`beacon-tools`),
   verified with record counts and checksums, then the facade routes that tenant to Beacon. A tenant can
   be routed back within minutes if needed.
6. **Decommission**: after the last wave, Helpdesk Classic is read-only for 90 days (for audits and
   exports), then archived and switched off. The 60% of unused routes were never migrated.

Notice what the plan avoids: no rewrite of Helpdesk Classic itself, no attempt to modernize it to .NET 10
(it will be switched off; investing in it would be waste), and no big-bang cutover.

> **🔍 Investigation:** During wave 2, a customer reports SLA breaches appearing on tickets that were
> fine in Helpdesk Classic. The parallel-run data shows the difference: the legacy stored procedure
> treated the customer's site time zone as fixed UTC+1 (ignoring daylight saving time), and their agents
> had adjusted their working hours around that bug for years. Fixing the "bug" broke their process. The
> team adds a per-tenant compatibility setting, documents it, and agrees a date with the customer to
> switch to correct time zone handling. Legacy behavior is often load-bearing.

---

## 8. What can go wrong

- **The big-bang rewrite** that takes three times as long as estimated and never reaches parity.
- **Modernizing what you should retire**: migrating features nobody uses.
- **Changing everything at once**: runtime, framework, ORM, hosting, architecture.
- **Losing behavior**: undocumented business rules in stored procedures, triggers, configuration and
  scheduled jobs that the new system doesn't reproduce.
- **Never finishing**: the strangler fig stalls at 80%, leaving two systems to maintain forever.
- **Ignoring knowledge risk**: the one person who understands the system leaves halfway through.
- **Debt as a moral argument** ("this code is bad") instead of a business one ("this costs us X per
  feature and caused Y incidents"), which fails to get time allocated.
- **Coverage targets** that produce tests without meaningful assertions.
- **Falling behind on platform versions** until the upgrade becomes a project instead of a routine.

---

## 9. When not to use it

> **🧭 When not to use it:** Not all legacy code needs attention. Code that is **stable, rarely changed,
> supported and secure** should usually be left alone, however old-fashioned it looks: refactoring it
> costs effort and adds risk for no benefit. Likewise, a system scheduled for retirement doesn't need
> modernization, only enough maintenance to stay secure until it's switched off. Spend improvement effort
> where the code is changing and where the business is going.

---

## 10. How an experienced engineer thinks about this

- **Measures maintainability by the cost and risk of change**, using delivery metrics and hotspots
  rather than opinions about code style.
- **Talks about debt in business terms**: interest paid, incidents caused, features delayed.
- **Pays debt continuously**, in the areas being changed, rather than in big cleanup projects.
- **Stays current with the platform** as a routine, so upgrades never become crises.
- **Gets legacy code under test before changing it**, using characterization tests and production data.
- **Prefers incremental migration** (strangler fig, branch by abstraction, parallel runs) to rewrites.
- **Respects legacy behavior**: it often encodes real business rules, and sometimes users depend on its
  bugs.
- **Retires as well as migrates**, and makes sure old systems actually get switched off.
- **Treats knowledge as an asset to protect**: pairing, documentation and spreading ownership.

---

## 11. Check yourself

**Questions**

1. What makes a system maintainable? Why is "time to safely ship a small change" a good indicator?
2. What are the four DORA metrics, and why do they reflect maintainability?
3. What's a hotspot, and why does complexity only matter where code changes?
4. Explain Fowler's technical debt quadrant. Which kinds of debt are legitimate?
5. Why do "tech debt sprints" often fail? What works better?
6. What's a characterization test? How do you handle bugs it pins?
7. What's a seam? Name three ways to introduce one.
8. Explain the strangler fig pattern and its advantages over a rewrite.
9. What's branch by abstraction? Parallel run?
10. Which .NET Framework technologies have no direct equivalent in modern .NET, and what are the paths
    for each?

**Exercises**

1. Run the churn analysis on a repository you work on and combine it with file size or complexity. Which
   three files are your hotspots? Do they match the team's intuition?
2. Write characterization tests with Verify for a piece of untested code, using realistic inputs. Then
   refactor it with automated refactorings only, keeping the tests green.
3. Start a debt register for a project: five items, each with evidence of interest and a rough cost.
4. Build a small strangler facade: an ASP.NET Core app with YARP forwarding to an existing app, with one
   route reimplemented. Add a parallel-run comparison for that route that logs differences.
5. If you have access to a .NET Framework application, do an assessment: project types, blockers,
   dependency order, and a migration plan of shippable steps.

**Interview-style questions**

- "How do you decide whether to refactor, rewrite or leave code alone?"
- "How would you convince management to invest in reducing technical debt?"
- "You've inherited a large codebase with no tests. Where do you start?"
- "How would you migrate a large ASP.NET MVC 5 application to modern .NET without stopping feature
  development?"
- "Tell me about a legacy system you improved. What would you do differently?"

---

## 12. Going deeper

- Michael Feathers, *Working Effectively with Legacy Code* — seams, characterization tests and dozens of
  dependency-breaking techniques; still the definitive book.
- Martin Fowler, *Refactoring* (2nd ed.), and the articles ["StranglerFigApplication"](https://martinfowler.com/bliki/StranglerFigApplication.html)
  and ["TechnicalDebtQuadrant"](https://martinfowler.com/bliki/TechnicalDebtQuadrant.html).
- Adam Tornhill, *Your Code as a Crime Scene* (2nd ed.) and *Software Design X-Rays* — hotspots, change
  coupling and knowledge maps from version control.
- Nicole Forsgren, Jez Humble and Gene Kim, *Accelerate* — the research behind the DORA metrics; and the
  annual [DORA reports](https://dora.dev).
- Microsoft, [Upgrade to .NET documentation](https://learn.microsoft.com/dotnet/core/porting/) and
  [incremental ASP.NET to ASP.NET Core migration](https://learn.microsoft.com/aspnet/core/migration/fx-to-core/).
- Marianne Bellotti, *Kill It with Fire* — a pragmatic, experience-based book on modernizing legacy
  systems.

**Next:** [How Experienced Engineers Think](08-how-experienced-engineers-think.md) pulls together the
judgment that runs through this whole book: approaching new projects, reading unfamiliar code and
choosing between technologies.
