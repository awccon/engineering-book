# Collaborating: Pull Requests and Code Review

Writing code is only part of a professional developer's job. Getting it reviewed,
reviewing others' code, and integrating changes safely into a shared codebase are where
team effectiveness is won or lost. A team with slow, hostile or rubber-stamp reviews
ships slower and with more bugs than a team with fast, thoughtful ones, regardless of
individual skill.

This chapter covers the pull request workflow, how to write a PR that's easy to review,
how to review well (including AI-generated code), and how branch strategies and
automation fit together.

---

## 1. The problem: many people, one codebase

As soon as two people work on the same code, you need answers to:

- How do changes get into the main line without breaking it?
- Who checks a change before it ships, and what do they check?
- How do we share knowledge so no part of the system is understood by only one person?

The **pull request** (PR; GitLab calls it a *merge request*) is the standard answer: a
proposal to merge a branch, with a discussion, automated checks and an approval step.

---

## 2. The mental model: a PR is a conversation with gates

```text
 feature branch ──push──► Pull request
                           ├─ Description: what and why
                           ├─ Diff
                           ├─ Automated checks: build, tests, lint, security scans
                           ├─ Review: comments, suggestions, approvals
                           └─ Merge (if all gates pass) ──► main ──► CI/CD deploys
```

Code review exists for several reasons, in roughly this order of value:

1. **Knowledge sharing**: more than one person understands every change.
2. **Design and maintainability**: is this the right approach? Will it be easy to change?
3. **Catching bugs**: logic errors, missed edge cases, security problems.
4. **Consistency**: conventions, naming, structure.

Notice that "style nits" isn't on the list. Formatting and simple conventions should be
enforced by tools, so humans can focus on what tools can't judge.

---

## 3. Writing a good pull request

The author's job is to make the reviewer's job easy. A reviewer's attention is the scarce
resource.

### Keep it small

Review quality drops sharply with size. A 50-line PR gets careful review; a 2,000-line PR
gets "LGTM." Aim for PRs that can be reviewed in under 30 minutes, typically under ~400
lines of meaningful change.

How to keep PRs small:

- **Split by layer or step**: refactoring first (no behavior change), then the feature.
- **Separate mechanical changes** (renames, formatting, package updates) from logic.
- **Ship incomplete features behind a feature flag**, in several PRs.
- **Stack PRs**: PR 2 builds on PR 1's branch.

### Write a description

```markdown
## What
Adds `Ticket.Escalate()` and an "Escalate" command in the CLI.

## Why
Support leads need to raise priority when a customer is blocked (BEA-142).
Escalation can't skip levels, so the audit trail shows each step.

## How
- Escalate raises priority one level; Urgent can't be escalated further.
- Closed tickets can't be escalated (DomainException, consistent with comments).
- TicketService.EscalateAsync returns Result<Ticket> with conflict/not_found errors.

## Testing
- Unit tests for each priority level, closed tickets and the service error paths.
- Ran the CLI manually: `beacon escalate 42`.

## Notes for reviewers
The SLA clock doesn't reset on escalation. Intentional? Open to discussion.
```

- **Link the issue or ticket.**
- **Explain decisions**, and flag what you're unsure about. Inviting scrutiny on the
  risky part gets better reviews than hoping nobody notices.
- **Include screenshots** for UI changes.
- **Review your own PR first.** Read the diff in the PR view before requesting review;
  you'll catch debugging code, commented-out blocks and accidental files.

### Respond well

- Treat comments as being about the code, not you.
- Reply to every comment: fix it, or explain why not. "Good catch, fixed in a3f2c1" or
  "I considered that, but X because Y; happy to change if you still prefer it."
- If a discussion goes back and forth more than twice, talk synchronously.

---

## 4. Reviewing well

### What to look for (in priority order)

1. **Correctness**: Does it do what the description says? Edge cases (null, empty, max
   values, concurrency, time zones)? Error handling?
2. **Design**: Is the approach right? Does it fit the architecture? Is anything in the
   wrong layer? Is it over- or under-engineered?
3. **Security**: Input validation, authorization checks, secrets, injection, logging of
   sensitive data (Book III, Chapter 9 has a checklist).
4. **Tests**: Do they test behavior? Do they cover the risky parts? Would they catch a
   regression?
5. **Readability**: Will the next developer understand this? Names, structure, comments
   explaining *why*.
6. **Operational concerns**: Logging, metrics, migrations, backward compatibility of APIs
   and database schemas, performance on production data volumes.

### How to comment

- **Be specific and explain why.** "This loads every ticket into memory; on production
  data that's ~2M rows. Could we filter in the query?" beats "inefficient."
- **Distinguish blocking from non-blocking.** Prefixes help: `nit:` (trivial, optional),
  `suggestion:`, `question:`, `blocking:`. Conventional Comments formalizes this.
- **Ask questions** when you're not sure: "What happens if two agents escalate at the same
  time?" invites thinking instead of defensiveness.
- **Praise good things.** "Nice use of the builder here" costs nothing and reinforces good
  patterns.
- **Suggest, don't rewrite.** GitHub's "suggestion" blocks let the author accept a small
  change with one click.
- **Review promptly.** A PR waiting two days blocks the author and invites merge
  conflicts. Many teams aim for a first response within one working day, or within hours.

### Approving

Approve when the change **improves the codebase** and has no blocking issues, even if it
isn't exactly how you'd have written it. Perfection isn't the standard; "better than before
and safe to ship" is.

---

## 5. Reviewing AI-generated code

AI coding assistants now write a large share of code in many teams. Reviewing that code
needs a slightly different mindset (Book XI, Chapter 11 goes deeper):

- **Plausible isn't correct.** AI-generated code is fluent and confident. It compiles, it
  looks idiomatic, and it can still be wrong in subtle ways: an off-by-one boundary, a
  missing `await`, a race condition, an API that doesn't exist in your version.
- **Check against the actual requirements**, not just whether the code is reasonable.
  The assistant didn't attend the planning meeting.
- **Verify APIs and packages.** Hallucinated methods, outdated APIs, and invented package
  names (a supply-chain risk: attackers register packages with commonly hallucinated
  names) all happen.
- **Look harder at security-sensitive code**: authentication, authorization, SQL,
  deserialization, cryptography, file paths.
- **Check the tests too.** Generated tests sometimes assert whatever the code currently
  does, including its bugs, or test mocks instead of behavior.
- **The author owns it.** "The AI wrote it" isn't an explanation in review. If the author
  can't explain why the code is correct, it isn't ready.

> **🧱 Durable:** The reviewer's core question doesn't change with who (or what) wrote the
> code: *"Do I understand this change well enough to be on call for it?"*

---

## 6. Automating the gates

Humans review what needs judgment; machines enforce everything else.

### Branch protection

On GitHub (Settings → Branches, or Rulesets) or Azure DevOps (branch policies), protect
`main`:

- Require pull requests; no direct pushes.
- Require one or more approvals; dismiss approvals when new commits are pushed.
- Require status checks (build, tests) to pass.
- Require branches to be up to date before merging (or use a **merge queue**, which tests
  each PR combined with the ones ahead of it).
- Require conversation resolution.
- Block force pushes and deletion.

### Automated checks in CI

A typical PR pipeline (Book IX, Chapter 8 builds one):

- **Build** and **test** (`dotnet build`, `dotnet test`).
- **Formatting and analyzers** (`dotnet format --verify-no-changes`; warnings as errors).
- **Security**: dependency vulnerability scanning (Dependabot, `dotnet list package
  --vulnerable`), secret scanning, static analysis (CodeQL).
- **Coverage report** (informational, not a hard gate).
- **Preview environments** for web apps.

### CODEOWNERS

A `CODEOWNERS` file automatically requests review from the right people:

```text
# .github/CODEOWNERS
/src/Beacon.Core/            @awccon
/src/Beacon.Infrastructure/  @awccon @data-team
/.github/workflows/          @platform-team
```

### PR templates

`.github/pull_request_template.md` pre-fills every PR description with the sections from
section 3.

---

## 7. Branching strategies, revisited

| | Trunk-based | GitHub flow | GitFlow |
|---|---|---|---|
| Long-lived branches | `main` only | `main` only | `main`, `develop` |
| Feature branch lifetime | Hours to ~2 days | Days | Days to weeks |
| Releases | Continuous, from `main` | Deploy on merge to `main` | Release branches |
| Incomplete work | Feature flags | Feature flags or long branches | Long feature branches |
| Best for | Web services with strong CI/CD | Most web apps and small teams | Versioned, scheduled releases |

**GitHub flow** (branch from `main` → PR → review → merge → deploy) is the pragmatic
default for most teams building web applications, and it's what this book uses. As CI
and tests mature, it naturally becomes trunk-based development.

> **🧭 When not to use GitFlow:** If you deploy a web service continuously, GitFlow's
> `develop` branch and release branches add ceremony and delay without benefit. It made
> sense for software shipped as versioned releases to customers; if that's not you, skip it.

---

## 8. In practice: Beacon's collaboration setup

Even working alone, setting up the workflow now pays off: CI catches mistakes, PRs record
decisions, and you practice the habits.

**1. A PR template:**

```markdown
<!-- .github/pull_request_template.md -->
## What

## Why

## How was this tested?

## Risks / notes for reviewers
```

**2. A CI workflow that runs on every PR:**

```yaml
# .github/workflows/ci.yml
name: CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'
      - run: dotnet restore
      - run: dotnet format --verify-no-changes
      - run: dotnet build --no-restore -c Release -warnaserror
      - run: dotnet test --no-build -c Release
```

**3. Branch protection** on `main`: require a PR, require the `build-and-test` check, block
force pushes. (Solo developers can allow themselves to approve or skip the approval
requirement, but keep the status check.)

**4. Open the PR** for `feature/ticket-escalation` from Chapter 2, fill in the template,
wait for CI, review your own diff, and squash-merge.

From here on, every substantial change to Beacon in this book is a good candidate for its
own branch and PR. It's the same workflow this book itself uses: updates to existing
chapters arrive as pull requests for review.

---

## 9. What can go wrong

- **Giant PRs** that can't be reviewed properly.
- **Rubber-stamp approvals** ("LGTM") on changes nobody understood.
- **Nitpick wars** over style that a formatter should settle.
- **Slow reviews** blocking work for days, encouraging bigger, riskier PRs.
- **Gatekeeping and hostility**, which drive people to avoid review or stop contributing.
- **Bypassing checks** with admin overrides "just this once."
- **Trusting AI-generated code** because it looks polished.

---

## 10. How an experienced engineer thinks about this

- **Optimize for the team's flow, not individual output.** Reviewing a teammate's PR
  promptly is often the highest-leverage thing you can do today.
- **Small PRs, fast reviews, frequent integration.** They reinforce each other.
- **Automate the objective; review the subjective.**
- **Review for understanding.** If you couldn't support this code in production, you
  haven't finished reviewing it.
- **Kind and direct are compatible.** Clear, specific, respectful feedback is a skill
  worth practicing.

---

## 11. Check yourself

**Questions**

1. Rank the purposes of code review. Why is knowledge sharing so high?
2. What makes a PR easy to review?
3. How should a reviewer distinguish blocking from non-blocking comments?
4. What extra risks should you look for when reviewing AI-generated code?
5. What branch protection rules would you set on `main`?
6. When is GitFlow appropriate, and when isn't it?

**Exercises**

1. Take a recent large change you made and plan how you could have split it into three
   to five smaller PRs.
2. Add the CI workflow and PR template to Beacon's repository and open a PR.
3. Review an open-source PR on a project you use. Write (but don't necessarily post) your
   comments using the priority order in section 4.
4. Ask an AI assistant to implement `TicketService.EscalateAsync`, then review its output
   using section 5's checklist.

**Interview-style questions**

- "What do you look for when reviewing code?"
- "How do you handle disagreement in a code review?"
- "Describe your team's branching and release workflow. What would you improve?"

---

## 12. Going deeper

- [Google's Engineering Practices: Code Review](https://google.github.io/eng-practices/review/)
- [Conventional Comments](https://conventionalcomments.org/)
- [GitHub docs: About protected branches](https://docs.github.com/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- *Accelerate* by Forsgren, Humble and Kim — the research behind small batches and
  trunk-based development.

---

## Book II wrap-up

Beacon now lives in Git, with a branch-and-PR workflow, CI checks and protected `main`.
That's the foundation for everything that follows: each new layer of Beacon arrives as
reviewed, tested changes.

**Next:** [Book III — .NET & Backend](../03-backend/README.md) turns Beacon into a web API.
