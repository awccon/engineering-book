# How Experienced Engineers Think

Every chapter in this book has ended with a section called "How an experienced engineer thinks about
this". This chapter collects the patterns behind those sections, the habits of judgment that separate
an engineer who knows many technologies from one who can be trusted with a system.

It's organized around four situations every engineer meets repeatedly: **starting something new**,
**understanding something that already exists**, **choosing between options** and **working when things
are uncertain**. It ends with how to keep learning in a field where the tools change every year and the
fundamentals barely change at all.

---

## 1. The problem: knowledge isn't judgment

You can know C#, SQL, React, Docker, Azure and LLM APIs in depth and still make poor decisions: building
the wrong thing well, choosing a fashionable technology that doesn't fit, rewriting a system that needed
a small fix, or spending a week on a problem a question to the right person would have solved in ten
minutes.

Experience mostly shows up not as knowing more answers, but as:

- asking **better questions** before starting;
- seeing **trade-offs** where others see a single right answer;
- knowing **what not to do** and what to leave alone;
- reducing **risk and uncertainty** early;
- **communicating** decisions so others can understand, challenge and build on them.

These skills can be learned deliberately. They're not talent; they're habits.

---

## 2. The mental model: engineering is decision-making under uncertainty

Software engineering is mostly a series of decisions with incomplete information: what to build, how to
structure it, which tools to use, what to test, when to ship, what to fix first. Experienced engineers
treat it that way explicitly.

> **🧱 Durable:** For every significant decision, an experienced engineer asks four things:
> 1. **What problem are we actually solving, and for whom?**
> 2. **What are the options, including doing nothing or doing less?**
> 3. **What does each option cost, now and later, and what does it make harder?**
> 4. **How reversible is it, and how will we know if it was wrong?**

Three ideas follow:

- **Reversibility determines how much analysis a decision deserves.** Two-way doors (an internal
  library, a folder structure, a UI layout behind a flag) should be decided quickly and changed if wrong.
  One-way doors (a public API, a data model that other teams build on, a database choice, a contract with
  a vendor) deserve careful thought, prototypes and written reasoning (Chapter 2's ADRs).
- **Reduce the biggest uncertainty first.** If you don't know whether the AI model can classify tickets
  accurately enough, build that evaluation before the UI. If you don't know whether the legacy API can
  handle your load, test that before designing around it.
- **Every choice has costs.** "Best practice" without context is a warning sign. The useful question is
  always "best for what, at what cost, in this situation?"

---

## 3. Approaching a new project

### Understand the problem before the solution

Most failed projects didn't fail technically. They built the wrong thing, or the right thing for the
wrong constraints. Before designing anything, find out:

- **Who are the users, and what are they trying to accomplish?** Not "a ticket system", but "support
  agents who handle 40 tickets a day and lose time switching between email and the CRM".
- **What does success look like, measurably?** "Median first-response time under 2 hours", "support
  can handle 30% more volume without hiring".
- **What are the constraints?** Deadline, budget, team size and skills, compliance requirements (data
  residency, audit), existing systems to integrate with, scale.
- **What's already been tried?** Previous attempts and why they failed are the most valuable information
  you'll get.
- **What's explicitly out of scope?** Agreeing on non-goals prevents scope creep and wasted design.

Talk to users, and watch them work if you can. Twenty minutes watching an agent use the current tools
will reveal more than a requirements document.

### Find the risks, then attack them

List what could make the project fail, then rank by likelihood and impact:

| Risk type | Beacon example | How to reduce early |
|---|---|---|
| Value: will people use it? | Will agents trust AI reply drafts? | Prototype with real tickets; measure acceptance rate |
| Feasibility: can we build it? | Can we classify ticket priority at 90% accuracy? | Evaluation dataset and a quick experiment (Book XI, Chapter 8) |
| Integration: will it connect? | Does the customer's SSO support the flows we need? | Spike against their test tenant in week one |
| Scale: will it hold? | 50M historical tickets to import | Load-test the import with realistic volume |
| Organizational: will it be adopted? | The support team has its own spreadsheets | Involve team leads in design; migrate their data |

Do the **riskiest things first**, while there's time to change course. Teams often do the opposite,
building the comfortable parts (login, CRUD screens, CI) first and discovering in month five that the
core assumption doesn't hold.

### Build a walking skeleton

A **walking skeleton** (Alistair Cockburn) or **tracer bullet** (*The Pragmatic Programmer*) is the
thinnest possible end-to-end implementation that touches every major component and is deployed to
production-like infrastructure: the React page calls the BFF, which calls the API, which writes to
PostgreSQL, with CI/CD, logging and a health check. It does almost nothing, but it works end to end.

That's how this book built Beacon: a working CLI in Book I, an API in Book III, a database in Book IV, a
UI in Book VI, deployment in Book IX, each time extending a system that already ran. The walking skeleton
surfaces integration problems early (auth configuration, networking, deployment permissions) and gives
every later feature a place to land.

### Write it down

For anything that takes more than a couple of weeks or affects other teams, write a short **design
document** before building: context, goals and non-goals, the proposed design, alternatives considered,
risks and open questions, and a rollout plan. Writing forces clarity; reviewers find problems while
they're cheap to fix; and the document explains decisions to people who join later.

Keep it short: two to six pages. A design document is a tool for thinking and agreeing, not a
deliverable.

---

## 4. Reading an unfamiliar codebase

You'll spend more of your career reading code than writing it, often under time pressure: a new job, an
incident in a system you don't own, an open-source library behaving strangely. Reading code is a skill
with techniques.

### Start from the outside

1. **Read the README and docs**, however outdated. Look for ADRs, architecture diagrams and runbooks.
2. **Build and run it.** If you can't run it locally in under an hour, that's your first finding (and a
   gift to the next person if you fix the setup instructions).
3. **Use it as a user.** Click through the main flows. Know what the system *does* before reading how.
4. **Look at the shape**: the solution and project structure, the dependency graph, the main entry
   points (`Program.cs`, route definitions, message handlers, scheduled jobs, `main.tsx`).

### Follow one request end to end

Pick one important flow (say, creating a ticket) and trace it through every layer: route → endpoint →
validation → service → domain → repository → SQL → events → handlers → response. Use the debugger, or
the trace view in your observability tool, which shows the real path including the parts that are hard
to find by reading (middleware, interceptors, filters, background handlers).

One flow, understood completely, teaches you the conventions of the whole codebase: how errors are
handled, how data access works, where validation lives, how things are named.

### Read the data model

Fred Brooks: "Show me your tables, and I won't usually need your flowcharts; they'll be obvious." The
database schema (or the main domain types) tells you what the system **is** more reliably than code. Look
at tables, relationships, constraints, and especially columns with names like `status`, `type` or
`legacy_flag`: they're where the business rules hide.

### Use history and tests

- **Git history** explains *why*: `git log -p --follow <file>` for a confusing file, `git blame` on a
  strange line (then read the PR it came from), the most-changed files (Chapter 7's hotspots).
- **Tests** are executable documentation of intended behavior, especially their names. Run them;
  change something and see what breaks.
- **Issue tracker and incidents**: the bugs a system has had tell you where it's fragile.

### Build a mental map, write it down

As you learn, sketch: the main components, how data flows, where state lives, which parts are risky. Keep
a running notes file of questions and answers. Then **share it**: a diagram you drew while onboarding is
often the best documentation the team has.

### Use AI assistants as a guide, not an oracle

AI coding tools (Book XI, Chapter 11) are very good at summarizing unfamiliar code, explaining a
framework's conventions, finding where something is implemented and drafting a map of a codebase. Use
them to speed up orientation, then **verify** important claims by reading the code or running it.
Assistants can confidently describe what code "probably" does based on naming patterns, and miss the
one line that does something unusual.

> **🔍 Investigation:** A useful exercise on your first week in any codebase: make a tiny, real change
> (fix a typo in an error message, add a missing test) and take it all the way to production. You'll
> learn the build, the review process, the pipeline, the environments and the deployment, which is
> knowledge no amount of reading gives you.

---

## 5. Deciding between technologies

You'll constantly choose: which library, which database, which framework, build or buy, which cloud
service. The tech industry's marketing makes every choice look urgent and every new tool look essential.
Experienced engineers are deliberately conservative, for good reasons.

### Choose boring technology, mostly

Dan McKinley's "Choose Boring Technology" argument: every company has a small number of **innovation
tokens** to spend. A new, unfamiliar technology costs far more than its learning curve: unknown failure
modes, immature tooling, little operational experience, fewer people to hire or ask. Spend tokens where
new technology gives a real competitive advantage, and use well-understood tools (PostgreSQL, .NET,
React, a mainstream cloud) for everything else.

"Boring" doesn't mean old or bad. It means **its failure modes are known**. PostgreSQL's weaknesses are
documented in thousands of blog posts; a database released last year has weaknesses nobody has found
yet.

Beacon spent its tokens carefully: Rust for two narrow components where its properties matter (Book XII),
and LLM features where they create real value (Book XI). Everything else is mainstream.

### Criteria that actually matter

When comparing options, the deciding factors are rarely the feature lists. In rough order of how often
they decide:

1. **Fit to the problem**: does it solve *this* problem at *this* scale, with *these* constraints?
2. **Team skills and hiring**: can this team build, debug and operate it at 3 a.m.?
3. **Operational cost**: what does it take to run, patch, monitor, back up and scale? Managed or
   self-hosted?
4. **Ecosystem and maturity**: documentation, libraries, community, support, track record in production.
5. **Total cost of ownership**: licensing, infrastructure, people's time, over years, not months.
6. **Reversibility and lock-in**: how hard is it to leave? What's the exit path?
7. **Integration with what you already run**: one more technology means one more set of skills, tools,
   dashboards and security patches.
8. **Longevity and governance**: who maintains it, how is it funded, what are its license terms (and
   might they change, as several popular projects' licenses did in recent years)?

Performance and features matter, but usually less than people expect: most mainstream options are fast
enough and capable enough for most applications.

### A decision process

1. **Write down the problem and constraints**, not the solution. "We need a vector database" is a
   solution; "we need semantic search over 20M chunks with tenant isolation, p95 under 200 ms" is a
   problem.
2. **Include the option of using what you already have**, and the option of doing less.
3. **Shortlist two or three options.** More than that and the comparison becomes superficial.
4. **Spike the riskiest questions** with real data, time-boxed (days, not weeks).
5. **Decide, and record the decision** in an ADR with the context, options and trade-offs.
6. **Set a review trigger**: "Revisit if the corpus exceeds 100M chunks or p95 exceeds 300 ms."

### In practice: does Beacon need a dedicated vector database?

Beacon stores embeddings in PostgreSQL with pgvector (Book XI, Chapter 5). A large customer will bring
20 million article and ticket chunks. A team member proposes moving vectors to a dedicated vector
database. Here's how the decision goes.

**Problem statement**: semantic and hybrid search over ~20M chunks (growing ~30% a year) with strict
tenant isolation, combined with keyword search and permission filters, p95 under 200 ms, kept consistent
with ticket and article changes.

**Options**: (A) stay on pgvector, tuned; (B) a dedicated vector database (managed); (C) Azure AI Search,
which combines vector and keyword search.

**Spike results** (three days, realistic data, illustrative numbers):

| Criterion | A: pgvector (HNSW, partitioned by tenant tier) | B: Dedicated vector DB | C: Azure AI Search |
|---|---|---|---|
| p95 latency, filtered hybrid query | 140 ms | 60 ms | 90 ms |
| Tenant isolation | RLS + filter in the same query | Namespaces per tenant; app-enforced | Filters/index per tenant; app-enforced |
| Consistency with source data | Transactional with ticket writes | Async sync pipeline needed | Async indexing pipeline needed |
| Keyword + vector hybrid | Yes (tsvector + vector in SQL) | Varies by product | Yes, built in |
| New operational surface | None | New service, backups, monitoring, security review | New service (managed) |
| Team experience | High | None | Low |
| Incremental monthly cost | ~Larger DB tier | Highest | Medium |

**Decision**: stay with pgvector (option A). It meets the latency target with headroom, keeps search
transactionally consistent and protected by the same row-level security, and adds no new system. Record
the review trigger: revisit if chunks exceed 100M or p95 exceeds 200 ms after tuning. Option C is the
likely next step if that happens, because hybrid search with built-in ranking would save work.

Notice that the fastest option didn't win. Latency was one requirement among several, and A already met
it. Consistency, security and operational simplicity decided.

> **🧭 When not to use it:** This kind of structured comparison is for choices that are costly to
> reverse. For a date-formatting library, a test assertion library or a CSS utility, pick a popular,
> maintained option quickly and move on. Spending a week comparing logging libraries is its own mistake.

---

## 6. Working with uncertainty

### Debugging like a scientist

Book III, Chapter 10 and Book VIII, Chapter 7 gave debugging methods for APIs and servers. The general
form is the scientific method:

1. **Observe**: gather facts. What exactly happens? When did it start? What changed? Who's affected?
2. **Hypothesize**: list possible explanations, not just the first one.
3. **Predict and test**: for each hypothesis, what would you see if it were true? Find the cheapest test
   that distinguishes them.
4. **Narrow down**: bisect (`git bisect`, binary search over configuration, removing half the input).
5. **Confirm the cause** by explaining every symptom, then fix, then verify the fix.

The most common debugging failure is **fixating on the first hypothesis** and spending hours proving it.
The second is **changing several things at once**, so you never learn which one mattered.

### Estimating honestly

Estimates are predictions under uncertainty, and should be presented that way:

- **Give ranges with confidence**, not single numbers: "3–6 weeks, most likely 4, assuming the SSO
  integration works as documented".
- **Break work down** until pieces are a few days each; uncertainty hides in large pieces.
- **Name the assumptions and unknowns**, and reduce the biggest ones with spikes.
- **Compare with past work**: how long did similar things actually take? (Reference-class forecasting.)
- **Re-estimate as you learn**, and communicate changes early. A slipped estimate reported in week two is
  a planning input; reported the day before the deadline, it's a crisis.

### Knowing when to ask

A common pattern in less experienced engineers is struggling alone for too long. A good rule of thumb:
time-box solo investigation (say, an hour or two for a blocker), then ask, with what you've tried and
learned. Asking well is a skill: context, what you expected, what happened, what you've ruled out.
Experienced engineers ask *more* questions than juniors, not fewer; they're just better questions.

---

## 7. Working with people

Most of an engineer's impact beyond a certain level comes through other people.

- **Write clearly.** Design docs, PR descriptions, ADRs, incident updates, and messages that can be read
  in thirty seconds. Lead with the conclusion or the request, then the supporting detail.
- **Explain trade-offs, not just choices.** "I recommend A because it keeps search transactional; the
  cost is higher DB load, which we can absorb up to 100M chunks" lets others engage with the reasoning.
- **Disagree and commit.** Argue your case clearly with evidence; once a decision is made, support it
  fully, and revisit it only when new information appears.
- **Review code to help, not to win** (Book II, Chapter 3): focus on correctness, design and risk;
  distinguish blocking issues from preferences; ask questions rather than issue verdicts.
- **Say no, or "not now", with reasons.** Every yes is a no to something else. Explaining the trade-off
  is more useful than either silent compliance or flat refusal.
- **Make others effective**: documentation, tooling, pairing, unblocking. A senior engineer who makes five
  people 20% more effective has more impact than one who writes the most code.

---

## 8. Learning continuously without drowning

This book has separated **durable** knowledge (🧱) from **current** knowledge (🔄) throughout, and that
distinction is the key to staying current without chasing every trend.

**Invest most in durable fundamentals.** How computers execute code, memory and concurrency, HTTP,
relational data and transactions, distributed systems' failure modes, security principles, design
trade-offs. These change slowly and make every new tool easier to learn, because new tools are mostly
new combinations of old ideas. Server-sent events in an LLM streaming API are Book III's HTTP; vector
indexes are Book IV's indexes with a different distance function; agents are Chapter 3's retries,
timeouts and idempotency with a model in the loop.

**Learn current tools just in time, deeply enough.** Learn the tool you need for the work in front of
you, properly, by building something real with it. Skim the rest so you know it exists and what problem
it solves.

**Have a filter for new technology.** When something new appears, ask: what problem does it solve? Is it
a problem I have? What does it replace, and what does it cost? Who is using it in production, and what
did they learn? Most new tools can be safely ignored for a year; the ones that matter will still be
there, more mature and better documented.

**Go to primary sources.** Official documentation, release notes, RFCs, papers and the source code. They
are more accurate than summaries, and reading them is a skill that compounds.

**Build things.** Reading creates familiarity; building creates understanding. Every book in this series
had a Beacon component for that reason, and Book XIV's projects exist for the same reason.

**Teach.** Explaining something (a blog post, a talk, a design review, onboarding a colleague) reveals
what you don't understand yet.

---

## 9. What can go wrong

- **Solution-first thinking**: choosing the technology before understanding the problem.
- **Resume-driven development**: choosing tools to learn them rather than to solve the problem.
- **Analysis paralysis** on reversible decisions, and **haste** on irreversible ones.
- **Ignoring the people side**: a technically excellent design nobody can operate or that the team
  doesn't understand.
- **Hero mode**: one person holding all the knowledge, working alone, being the bottleneck.
- **Cargo culting**: copying practices from very different organizations ("Netflix does it") without
  their context.
- **Overconfidence in estimates** and in first hypotheses.
- **Chasing every trend**, or the opposite, refusing to learn anything new.

---

## 10. When not to use it

> **🧭 When not to use it:** Process is a tool, scaled to the stakes. A one-person weekend project
> doesn't need a design document, a risk register and ADRs. An afternoon bug fix doesn't need a
> technology comparison matrix. Over-applying deliberate decision-making slows teams as surely as
> under-applying it. The skill is noticing which decisions matter: expensive, irreversible, or affecting
> many people. Give those your full attention, and make the rest quickly.

---

## 11. How an experienced engineer thinks about this

Since this whole chapter is the answer, here it is condensed into habits:

- **Problem before solution.** Ask who it's for, what success means, and what the constraints are.
- **Risk first.** Attack the biggest uncertainty while it's cheap to change course.
- **End to end early.** A walking skeleton beats perfect components that have never met.
- **Reversibility sets the pace.** Decide two-way doors fast, one-way doors carefully.
- **Boring by default.** Spend innovation tokens only where they create real advantage.
- **Read before writing.** Run it, trace one flow, read the data model and the history.
- **Hypotheses, not hunches.** Debug and estimate with evidence and ranges.
- **Write it down.** Decisions, designs and learnings, briefly and clearly.
- **Fundamentals compound.** Learn the durable ideas deeply and the current tools as needed.
- **Impact through others.** Clear communication, kind review and shared knowledge scale beyond what one
  person can build.

---

## 12. Check yourself

**Questions**

1. What four questions should you ask about any significant decision?
2. What's the difference between a one-way and a two-way door decision? Give two examples of each from
   Beacon.
3. Why should a project attack its riskiest assumptions first? Give an example.
4. What's a walking skeleton, and what problems does it reveal early?
5. Describe a method for understanding an unfamiliar codebase in your first week.
6. What does "choose boring technology" mean, and what are innovation tokens?
7. List the criteria that most often decide technology choices. Why do feature lists rarely decide?
8. In the vector database example, why didn't the fastest option win?
9. How should you present an estimate?
10. How do you decide what to learn next in a fast-moving field?

**Exercises**

1. Write a two-page design document for a Beacon feature you'd like to add, including goals, non-goals,
   alternatives and risks. Ask someone to review it.
2. Pick an open-source .NET or TypeScript project you've never seen. Within two hours, trace one request
   or operation end to end and draw a diagram of its main components. Check your understanding by making
   a small change and running its tests.
3. Take a technology decision your team made in the past year and write the ADR that should have been
   written. Was the decision right in hindsight? What would have changed it?
4. For your next task, write an estimate as a range with assumptions. Afterwards, compare with what
   actually happened.
5. Make a personal learning plan for the next six months: two durable topics to deepen, two current tools
   to learn by building something, and what you'll deliberately ignore.

**Interview-style questions**

- "How do you approach a project where the requirements are unclear?"
- "Tell me about a technology choice you made. How did you decide, and how did it turn out?"
- "You join a team with a large, unfamiliar codebase. How do you get productive?"
- "Describe a time you disagreed with a technical decision. What did you do?"
- "How do you keep your skills current?"

---

## 13. Going deeper

- Andrew Hunt and David Thomas, *The Pragmatic Programmer* (20th anniversary ed.).
- Will Larson, *Staff Engineer* and *An Elegant Puzzle* — how senior engineers work and make decisions
  in organizations.
- Tanya Reilly, *The Staff Engineer's Path*.
- Dan McKinley, ["Choose Boring Technology"](https://boringtechnology.club).
- Fred Brooks, *The Mythical Man-Month* (especially "No Silver Bullet").
- Steve McConnell, *Software Estimation: Demystifying the Black Art*.
- Annie Duke, *Thinking in Bets* — decision-making under uncertainty, from outside software.
- Camille Fournier, *The Manager's Path* — useful even if you never manage, to understand how engineering
  organizations work.

**Next:** [System Design and Interviews](09-system-design-and-interviews.md) turns everything in this book
into a repeatable method for designing systems, and for demonstrating that skill in interviews.
