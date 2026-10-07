# Using AI as a Developer

The rest of Book XI is about building AI into products. This chapter is about using AI to
build products: coding assistants in the editor, chat assistants for design and debugging,
and agentic tools that read a repository, run commands, write code and open pull requests.
They've changed day-to-day software development as much as any tool in decades.

They also create new failure modes: plausible code that's subtly wrong, security holes copied
in at speed, dependencies that don't exist, and developers who stop understanding their own
codebase. This chapter covers how to get the benefits while keeping your judgment, your code
quality and your skills.

---

## 1. The problem: speed without understanding

AI coding tools can produce in seconds what took an hour: boilerplate, tests, data mappings,
migrations, scripts, documentation, a first draft of a feature. That speed is real. But:

- The code looks right, compiles and passes the obvious test, and still has an off-by-one
  boundary, a race condition, a missing authorization check or an N+1 query.
- The tool doesn't know your architecture decisions, your security requirements or why the
  codebase looks the way it does, unless you tell it.
- **You** are accountable for every line you commit, regardless of who or what wrote it (Book II,
  Chapter 3).

The skill is no longer only writing code; it's **directing, verifying and integrating** code.

---

## 2. The mental model: a fast, well-read, unreliable collaborator

Think of an AI coding assistant as a collaborator who:

- has read an enormous amount of code and documentation (up to a cutoff),
- types extremely fast and never gets tired,
- is often right, sometimes brilliantly so,
- is sometimes confidently wrong, invents APIs, and misses context,
- doesn't know your codebase beyond what it can see or search,
- doesn't carry responsibility for the outcome.

That's exactly the colleague you'd **pair with and review carefully**, give clear specifications to,
and verify with tests. Everything in this book about clear specifications (Chapter 3), verification
(Chapter 8) and code review (Book II, Chapter 3) applies.

### Kinds of tools

| Tool type | What it does | Good for |
|---|---|---|
| **Inline completion** | Suggests the next lines as you type | Boilerplate, repetitive patterns, obvious continuations |
| **Chat in the IDE** | Answers questions with your open files as context; edits selected code | Explaining code, small refactors, writing tests, debugging help |
| **Agentic coding tools** (in the IDE or terminal) | Read and search the repo, edit many files, run builds/tests/commands, iterate on failures | Multi-file features, migrations, test suites, investigations, PR preparation |
| **Cloud/background agents** | Work on tasks asynchronously in a sandbox and open pull requests | Well-scoped issues, dependency upgrades, bulk changes |
| **Review assistants** | Comment on pull requests | A first pass of review before humans |

> **🔄 Current (as of October 2026):** Widely used tools include GitHub Copilot (completion, chat,
> agent mode and a cloud coding agent), Claude Code, Cursor, Windsurf, JetBrains AI, and OpenAI's
> Codex, among others. Capabilities converge quickly; the practices in this chapter apply to all of them.

---

## 3. Working effectively with coding agents

### Give context and constraints

Agents do much better with explicit project knowledge. Most tools read a project instructions file
(names vary: `CLAUDE.md`, `AGENTS.md`, `.github/copilot-instructions.md`, `.cursor/rules`). Put there:

- architecture and layering rules ("Beacon.Core must not reference EF Core or ASP.NET Core"),
- conventions (minimal APIs, `Result<T>` for expected failures, `TimeProvider` for time),
- how to build, test and lint (`dotnet test`, `pnpm --dir web test`),
- what not to do ("never commit secrets; never disable nullable warnings; don't add packages without asking").

This book's own repository has such a file guiding how chapters are written: the same idea.

### Specify the task like a ticket

A good request has a goal, constraints, acceptance criteria and verification:

```text
Add an Escalate operation to tickets.

- Domain: Ticket.Escalate(DateTimeOffset now) raises priority one level; Urgent can't be escalated;
  closed tickets throw DomainException; raises a TicketEscalated domain event.
- Application: TicketService.EscalateAsync returns Result<Ticket> (not_found, conflict).
- API: POST /api/tickets/{id}/escalate, Staff policy, 404/409 problem details.
- Tests: unit tests for every priority and the closed case; one API integration test.
- Follow the existing patterns in Ticket.Resolve and the comment endpoint.
- Run dotnet build and dotnet test; fix failures before finishing.
```

That's Chapter 3's prompt engineering applied to your own work.

### Work in small, verifiable steps

- Ask for a **plan first** for anything non-trivial; review the plan before code is written.
- Prefer **small diffs** you can review in minutes, like small PRs (Book II, Chapter 3).
- Have the agent **run tests and linters** and iterate on failures; tests are the agent's feedback loop.
- **Commit at good checkpoints** so you can roll back an agent's detour.
- For investigations ("why is this endpoint slow?"), ask for **evidence**: the query plan, the trace,
  the exact lines, not just a conclusion (Book IV, Chapter 6).

### Where agents shine, and where to be careful

| Usually great | Be careful |
|---|---|
| Boilerplate, DTOs, mappings, CRUD endpoints following existing patterns | Security-sensitive code (auth, crypto, input handling) |
| Tests for existing code, especially edge cases you list | Concurrency, distributed systems, transactions |
| Explaining unfamiliar code and libraries | Performance-critical paths |
| Mechanical refactors across many files | Database migrations on production data |
| Scripts, data transformations, one-off tools | Anything you don't understand well enough to review |
| First drafts of docs, ADRs, PR descriptions | Architecture decisions (use AI as a sounding board, decide yourself) |

---

## 4. Reviewing AI-generated code

Book II, Chapter 3 introduced the checklist; here it is in depth. Review AI code **at least as
carefully** as a new colleague's, with extra attention to these failure patterns:

| Failure pattern | What to check |
|---|---|
| **Hallucinated APIs** | Methods, options or packages that don't exist or belong to another version |
| **Outdated patterns** | Deprecated APIs, old framework idioms (e.g. `Startup.cs` for new ASP.NET Core apps) |
| **Plausible logic errors** | Boundaries (`>` vs `>=`), null handling, time zones, rounding, off-by-one in pagination |
| **Missing authorization** | Endpoints that load by ID without checking ownership (Book III, Chapter 6) |
| **Injection** | String-built SQL, unvalidated paths, unescaped output |
| **Swallowed errors** | `catch { }`, ignored results, fire-and-forget tasks (Book I, Chapters 8–9) |
| **Concurrency** | Shared mutable state in singletons, `.Result`, missing cancellation |
| **Performance** | N+1 queries, loading whole tables, unbounded lists |
| **Tests that test nothing** | Assertions that mirror the implementation, excessive mocking, tests asserting buggy behavior |
| **Architectural drift** | Domain code referencing infrastructure, new patterns inconsistent with the codebase |
| **Unnecessary dependencies** | A package added for one trivial function |

### Supply-chain caution: slopsquatting

Models sometimes suggest packages that don't exist. Attackers register those names with malicious code
("slopsquatting"). Before adding any dependency suggested by AI:

- Confirm it exists on the official registry, is the project you think it is (publisher, repository,
  download counts, age), and is maintained.
- Prefer the platform's built-in capabilities and well-known packages.
- Lock files and dependency review in CI (Book V, Chapter 3; Book X, Chapter 2) catch some, not all.

### The understanding test

Before approving AI-written code (yours or a teammate's), you should be able to answer:

- What does this do, and why this way?
- What happens on failure, on concurrency, on bad input, on large data?
- How would I debug it at 3 a.m.?

If not, it isn't ready, however good it looks.

---

## 5. Security and confidentiality when using AI tools

- **Know where your code goes**: enterprise plans of major tools generally offer data protection terms
  (no training on your code, limited retention), but check your organization's policy and the tool's
  settings. Don't paste proprietary code or customer data into consumer tools that don't offer those
  guarantees.
- **Never paste secrets** (connection strings, keys, tokens) into prompts. If one appears in code the
  tool reads, rotate it (and it shouldn't be in code anyway).
- **Agents with shell access** can run commands: run them in containers, dev containers or sandboxes
  where possible; review commands before allowing destructive or networked operations; don't give them
  production credentials.
- **Prompt injection applies to coding agents too**: a malicious README, issue comment or dependency
  file can contain instructions for the agent (Chapter 9). Be careful when agents process untrusted
  repositories or issues.
- **Licensing**: some tools offer filters for code matching public repositories; follow your
  organization's policy on generated code and licenses.

---

## 6. Keeping your skills

There's a real risk: if AI writes everything, developers stop building the mental models that let them
review, debug and design. Ways to stay sharp:

- **Understand before you accept.** Read the generated code; ask the tool to explain parts you don't
  understand; check against documentation.
- **Write the hard parts yourself sometimes**, especially when learning: the domain model, the tricky
  query, the concurrency fix. Then compare with what an assistant would do.
- **Use AI as a tutor**: "explain why this deadlocks," "what are the trade-offs of these two designs,"
  "quiz me on EF Core change tracking." This book's "Check yourself" questions work well this way.
- **Debug without it periodically**, using the methods from Book III, Chapter 10 and Book VIII,
  Chapter 7. When the AI is wrong, you'll need these skills.
- **Own the design.** Use AI to explore options; make and document decisions yourself (ADRs, Book XIII).

> **🧱 Durable:** Tools change every few months; fundamentals compound for decades. The engineers who
> get the most from AI tools are the ones who could do the work without them, and therefore know what
> to ask for and how to check the answer.

---

## 7. AI in interviews and the job market

Many companies now expect fluency with AI tools, and interviews increasingly include working with them
(or explicitly without them). Be ready to:

- explain how you use AI tools and how you verify their output,
- solve problems without assistance, explaining your reasoning,
- review a piece of (possibly AI-generated) code and find its problems,
- discuss AI application design: RAG, evaluation, prompt injection, cost (this book).

Book XIII, Chapter 9 covers interview preparation in depth.

---

## 8. In practice: an AI-assisted workflow on Beacon

A realistic feature, "ticket watchers" (users can follow tickets and get notified), done with an agentic
coding tool, following this chapter:

1. **Context**: `AGENTS.md`/`CLAUDE.md` in the repo describes Beacon's architecture, layering rules, test
   commands and conventions (as this book's repo does for chapter writing).
2. **Plan**: ask for a plan covering schema (`ticket_watchers` table, Book IV, Chapter 1 exercise), domain
   events, API endpoints, notifications via the outbox, frontend toggle and tests. Review it: the agent
   proposed sending notifications directly from the endpoint; redirect it to the outbox (Book III,
   Chapter 8).
3. **Implement in slices**, each a small commit: migration + entity; domain + unit tests; endpoints +
   integration tests; outbox handler; frontend + component tests. The agent runs `dotnet test` and
   `pnpm test` after each slice.
4. **Review** each slice with section 4's checklist. Findings in this example: the "list my watched tickets"
   endpoint didn't filter by tenant (missing authorization; fixed and a test added); a hallucinated EF Core
   method (`ExecuteUpsertAsync`) replaced with `on conflict` SQL; a test that asserted the mock was called
   replaced with a state assertion (Book I, Chapter 14).
5. **Contract and E2E**: OpenAPI diff reviewed (Book VII, Chapter 2); one Playwright journey added.
6. **PR**: description written with AI help, edited by you, including what was AI-generated and how it was
   verified. Human review by a teammate as usual.

Result: the feature took a fraction of the usual time, and the review caught three issues that would have
reached production. The tool provided speed; the engineering practices from this book provided correctness.

---

## 9. What can go wrong

- **Accepting code you don't understand.**
- **Huge AI-generated PRs** that can't be reviewed properly.
- **Hallucinated APIs and packages**, including malicious lookalikes.
- **Security regressions** (missing authorization, injection) introduced quickly and widely.
- **Tests that pass but verify nothing.**
- **Leaking secrets or proprietary code** to tools without appropriate terms.
- **Agents running destructive commands** or with production credentials.
- **Skill atrophy**: losing the ability to debug and design without the tool.

---

## 10. How an experienced engineer thinks about this

- **AI writes drafts; engineers own outcomes.**
- **Specify clearly, work in small verified steps, review rigorously.**
- **Tests and CI are the safety net** for human and AI code alike.
- **Protect secrets, data and environments** from tools that don't need them.
- **Keep learning the fundamentals**; they're what make AI tools useful in your hands.

---

## 11. Check yourself

**Questions**

1. What kinds of AI coding tools exist, and what is each best at?
2. What should a project instructions file for coding agents contain?
3. Why work in small, verified steps with agents?
4. Name eight failure patterns to check for in AI-generated code.
5. What is slopsquatting, and how do you protect against it?
6. What security precautions apply when coding agents can run commands?
7. How can you use AI tools without losing your own skills?

**Exercises**

1. Write an `AGENTS.md` (or equivalent) for Beacon describing architecture, conventions and commands.
2. Use a coding agent to implement `Ticket.Escalate` from section 3's specification; review the result with
   section 4's checklist and record what you found.
3. Ask an AI assistant to write a function with a known subtle requirement (e.g. SLA boundary at exactly 60
   minutes) and test whether its output handles it.
4. Pick a "Check yourself" question from an earlier chapter, answer it yourself, then ask an AI tutor to
   critique your answer.

**Interview-style questions**

- "How do you use AI tools in your development work?"
- "How do you review AI-generated code?"
- "What risks do AI coding assistants introduce, and how do you manage them?"

---

## 12. Going deeper

- Anthropic, ["Claude Code best practices"](https://www.anthropic.com/engineering/claude-code-best-practices)
- [GitHub Copilot documentation](https://docs.github.com/copilot)
- Simon Willison's writing on using LLMs for code (simonwillison.net).
- OWASP guidance on AI-assisted development and software supply chain security.

---

## Book XI wrap-up

You now have a working model of LLMs and the full toolkit of AI application engineering: calling models
reliably, prompts as specifications, structured output and tools, embeddings and hybrid search, RAG with
citations and permissions, agents with budgets and human approval, evaluation that measures quality and
manages hallucinations, defenses against prompt injection and data leakage, cost control, and a
full-stack architecture in .NET and React. And you know how to use AI tools to build software while
keeping ownership of the result.

Beacon has AI triage, summaries, reply drafts, a grounded help assistant and an incident agent, all
evaluated, secured, observable and behind flags.

**Next:** [Book XII — Rust](../12-rust/README.md) looks at a very different approach to building software:
safety and performance without a garbage collector.
