# Branching, Merging and Rebasing

Branches let you work on several things at once: a feature, a bug fix and an experiment,
each isolated from the others and from the stable code. The hard part is bringing them
back together. That's where merge conflicts, "merge vs rebase" debates and tangled
histories come from.

With the model from Chapter 1 (commits are snapshots in a graph; branches are labels),
merging and rebasing are just two different ways of combining lines of history, each with
clear trade-offs.

---

## 1. The problem: parallel lines of work

You start a feature branch from `main`. While you work, teammates merge other changes
into `main`. Now there are two lines of history that diverged from a common ancestor:

```text
          D ◄── E          feature
         ╱
 A ◄── B ◄── C             main
```

Eventually your work must be combined with theirs. There are two fundamental ways to do
it.

---

## 2. Merging

```bash
git switch main
git merge feature
```

Git finds the **merge base** (the most recent common ancestor, `B`), computes what changed
from `B` to `C` (theirs) and from `B` to `E` (yours), combines both sets of changes, and
records the result as a new **merge commit** `M` with two parents:

```text
          D ◄── E
         ╱       ╲
 A ◄── B ◄── C ◄── M     main
```

Properties of merging:

- **Non-destructive.** Existing commits are untouched. Nothing is rewritten, so it's always
  safe on shared branches.
- **History shows what really happened**, including that work happened in parallel.
- **History can get noisy**: many small merge commits ("Merge branch 'main' into feature")
  make `git log --graph` hard to read.

### Three-way merge

Git merges with three inputs: the base, ours and theirs. For each region of each file:

| Base | Ours | Theirs | Result |
|---|---|---|---|
| X | X | X | X |
| X | Y | X | Y (only we changed it) |
| X | X | Z | Z (only they changed it) |
| X | Y | Y | Y (both made the same change) |
| X | Y | Z | **Conflict**: both changed the same region differently |

This is why most merges are automatic: conflicts only happen when *both sides changed the
same lines* (or one side deleted a file the other modified).

---

## 3. Rebasing

```bash
git switch feature
git rebase main
```

Rebase **replays** your commits on top of the latest `main`: it takes the changes from
`D` and `E` and re-applies them, one by one, as new commits `D'` and `E'` whose parent is
`C`:

```text
 A ◄── B ◄── C ◄── D' ◄── E'     feature
                ▲
              main
```

`D` and `E` still exist (in the reflog) but no branch points to them anymore. `D'` and
`E'` have the same changes but **different IDs** (different parents → different hashes).

Now `main` can fast-forward to `E'`, giving a **linear history** with no merge commit.

Properties:

- **Clean, linear history** that's easy to read and bisect.
- **Rewrites history.** The rebased commits are new commits.
- **Conflicts are resolved per commit**, which can mean resolving similar conflicts
  several times on a long branch.

### The golden rule of rebasing

> **⚠️ What can go wrong:** **Never rebase commits that others have based work on.** If
> you rebase a branch that a colleague has pulled, their copy still has `D` and `E`, while
> yours has `D'` and `E'`. When they pull, Git sees two divergent histories with duplicate
> changes. Rebase your own local or private feature branches freely; don't rebase `main`
> or shared branches.

### Interactive rebase: cleaning up before sharing

`git rebase -i` lets you rewrite your branch's commits before opening a pull request:

```text
pick   a1b2c3 Add triage queue
fixup  d4e5f6 fix typo
pick   0718aa Add triage tests
reword 9b1c2d wip
drop   5e6f70 debug logging
```

- `squash`/`fixup`: combine commits.
- `reword`: change a message.
- `edit`: stop to amend a commit (split it, change it).
- `drop`: remove a commit.
- Reorder lines to reorder commits.

A useful workflow: commit messily while working (`fixup!` commits via
`git commit --fixup <sha>`), then `git rebase -i --autosquash main` to fold them in.

---

## 4. Merge vs rebase: how to decide

This is one of the most debated topics in software teams. A pragmatic summary:

| | Merge | Rebase |
|---|---|---|
| Safety on shared branches | Always safe | Only for unshared commits |
| History | True, branchy | Clean, linear |
| Conflict resolution | Once, at merge time | Per replayed commit |
| `git bisect` / reading history | Harder with many merge commits | Easy |
| Preserves context of parallel work | Yes | No |

Common team conventions:

1. **Rebase your feature branch on `main` locally** to stay current (`git pull --rebase`,
   or `git rebase main`) while it's private.
2. **Integrate into `main` through a pull request**, using one of:
   - **Merge commit**: keeps all branch commits plus a merge commit.
   - **Squash merge**: combines the whole branch into **one** commit on `main`. Simple
     history; individual commits of the branch are lost.
   - **Rebase merge**: replays the branch commits onto `main`, linear, no merge commit.

Many teams default to **squash merge** for small feature branches: one PR = one commit on
`main`, easy to revert. Teams that care about granular commits use rebase merge.

> **🧭 When not to rebase:** If a branch is shared, long-lived, or you're unsure whether
> anyone has pulled it, merge. A slightly messier history is much cheaper than a
> confused team and duplicated commits.

Set the default for `git pull` so you don't create accidental merge commits from your own
remote branch:

```bash
git config --global pull.rebase true
```

---

## 5. Resolving conflicts

When both sides changed the same region, Git stops and marks the conflict in the file:

```text
<<<<<<< HEAD
    public static TimeSpan ResponseTarget(TicketPriority p) => p switch
    {
        TicketPriority.Urgent => TimeSpan.FromMinutes(30),
||||||| base
    public static TimeSpan ResponseTarget(TicketPriority priority) => priority switch
    {
        TicketPriority.Urgent => TimeSpan.FromHours(1),
=======
    public static TimeSpan ResponseTarget(TicketPriority priority) => priority switch
    {
        TicketPriority.Urgent => TimeSpan.FromHours(2),
>>>>>>> feature/relax-sla
```

(The `||||||| base` section appears if you set `git config --global merge.conflictstyle
zdiff3`, which is highly recommended: seeing the *original* tells you what each side
intended.)

Here, one side renamed the parameter *and* changed urgent to 30 minutes; the other changed
it to 2 hours. Resolving it isn't a text problem; it's a **product question**: what is the
correct SLA? Talk to the other author if you aren't sure.

The process:

1. `git status` lists conflicted files.
2. Edit each file to the correct result; remove the markers.
3. **Build and run the tests.** A merge with no textual conflicts can still be
   semantically broken (one side renamed a method, the other added a call to the old name).
4. `git add` the resolved files, then `git merge --continue` (or `git rebase --continue`).
5. If it's going badly: `git merge --abort` / `git rebase --abort` returns to the state
   before you started.

### Reducing conflicts

- **Integrate often.** Small, short-lived branches rebased or merged frequently conflict
  rarely, and when they do, the conflicts are small.
- **Avoid sweeping reformatting** in feature branches. Do formatting changes as separate
  PRs, ideally with an automated formatter (`dotnet format`) enforced in CI.
- **Coordinate on hot files** (a central `Program.cs`, a shared enum).
- **`git rerere`** ("reuse recorded resolution"): `git config --global rerere.enabled true`
  makes Git remember how you resolved a conflict and reapply it automatically next time.

---

## 6. Other ways to move commits

- **`git cherry-pick <sha>`**: copy one commit onto the current branch. Useful for
  backporting a fix to a release branch. Creates a new commit with the same changes.
- **`git revert <sha>`**: create a commit that undoes an earlier commit. The safe way to
  undo something on `main`. Reverting a merge commit needs `-m 1` to choose which parent
  is "mainline."
- **`git stash`**: set aside uncommitted work to switch tasks; `git stash pop` brings it
  back. Prefer a quick WIP commit on a branch for anything longer than a few minutes;
  stashes are easy to forget.
- **`git worktree add ../beacon-hotfix hotfix/123`**: check out a *second* branch in
  another directory, sharing the same repository. Ideal for reviewing a PR or fixing a
  production bug without disturbing your current work.

---

## 7. Branch strategies at a glance

Teams organize branches in a few standard ways. Chapter 3 discusses them with the
collaboration workflow, but the core idea is:

- **Trunk-based development**: everyone integrates into `main` at least daily through
  short-lived branches (hours to a couple of days). Incomplete features hide behind
  feature flags. Requires good CI and tests. Favored by high-performing teams and by
  continuous delivery.
- **GitFlow**: long-lived `develop` and `main` branches plus feature, release and hotfix
  branches. Designed for versioned, scheduled releases (packaged software). Heavy for web
  services deployed continuously.
- **Release branches**: `main` plus `release/x.y` branches for maintaining versions in
  production, with fixes cherry-picked back.

> **🧱 Durable:** The longer a branch lives, the more it diverges, and the more expensive
> integration becomes. Every branching strategy is a trade-off between isolation and the
> cost of integrating later. Short-lived branches minimize that cost.

---

## 8. In practice: a feature branch for Beacon

Let's do a complete cycle with a realistic conflict.

```bash
git switch -c feature/ticket-escalation
```

Implement `Ticket.Escalate()` (Book I, Chapter 3 exercise) and its tests, committing as
you go:

```bash
git add src/Beacon.Core/Tickets/Ticket.cs
git commit -m "Add Ticket.Escalate to raise priority one level"
git add tests/Beacon.Core.Tests/Tickets/TicketTests.cs
git commit -m "Test escalation rules"
```

Meanwhile, simulate a teammate's change on `main`:

```bash
git switch main
# edit Ticket.cs: add a Description property next to Title
git commit -am "Add ticket description"
git switch feature/ticket-escalation
```

Bring your branch up to date by rebasing (it's private, so that's safe):

```bash
git fetch origin            # in a real team; here main is local
git rebase main
```

If both changes touched neighboring lines in `Ticket.cs`, you'll get a conflict. Resolve
it to include *both* the `Description` property and `Escalate`, then:

```bash
dotnet build && dotnet test
git add src/Beacon.Core/Tickets/Ticket.cs
git rebase --continue
git log --oneline --graph --all
```

The graph is now linear: your two commits sit on top of the teammate's. Tidy up if needed
(`git rebase -i main`), push the branch, and open a pull request, which is where the next
chapter begins:

```bash
git push -u origin feature/ticket-escalation
```

---

## 9. What can go wrong

- **Rebasing shared branches**, creating duplicated commits for everyone else.
- **Force-pushing over others' work.** Always `--force-with-lease`.
- **Resolving conflicts textually without understanding them.** Keep both sides blindly,
  and the code compiles but behaves wrongly.
- **Not building/testing after a merge.** Semantic conflicts don't show up as conflict
  markers.
- **Long-lived branches** that become painful, risky "big bang" merges.
- **Accidental merge commits** from `git pull` on a branch with local commits.

---

## 10. How an experienced engineer thinks about this

- **Integrate early and often.** Small branches, merged quickly, are the single best
  conflict-prevention technique.
- **Rebase what's private; merge what's shared.**
- **Conflicts are communication problems.** Two people changed the same thing; often the
  right fix is a conversation, not an edit.
- **Choose a team convention and automate it** (protected branches, merge strategy in PR
  settings) instead of debating it on every PR.

---

## 11. Check yourself

**Questions**

1. What's a merge base, and how does Git use it in a three-way merge?
2. What does a rebase do to commit IDs? Why?
3. What's the golden rule of rebasing, and what goes wrong if you break it?
4. Compare merge commit, squash merge and rebase merge for integrating a PR.
5. Why should you build and test after a conflict-free merge?
6. What does `git rerere` do?

**Exercises**

1. Create a conflict deliberately, set `merge.conflictstyle zdiff3`, and resolve it.
2. Make five messy commits, then use `git rebase -i` to turn them into two clean ones.
3. Cherry-pick a fix from a feature branch onto a `release/1.0` branch.
4. Use `git worktree` to check out a second branch while keeping your current work open.

**Interview-style questions**

- "What's the difference between merge and rebase? When do you use each?"
- "How do you resolve a merge conflict?"
- "What branching strategy has your team used, and what would you change?"

---

## 12. Going deeper

- [*Pro Git*, chapter 3: Git Branching](https://git-scm.com/book/en/v2/Git-Branching-Branches-in-a-Nutshell)
- [Atlassian: Merging vs. rebasing](https://www.atlassian.com/git/tutorials/merging-vs-rebasing)
- [trunkbaseddevelopment.com](https://trunkbaseddevelopment.com/)

**Next:** [Chapter 3 — Collaborating: Pull Requests and Code Review](03-collaborating-pull-requests-and-code-review.md)
covers how teams review and integrate each other's work.
