# How Git Actually Works

Most developers learn Git as a set of commands: `add`, `commit`, `push`, `pull`, and a
few others looked up when something goes wrong. That works until it doesn't: a detached
HEAD, a rebase that "lost" commits, a merge conflict mid-way through a pull, a force-push
that overwrote a colleague's work. At that point, memorized commands don't help.

Git's internal model is small and elegant. Once you understand it, almost every command
becomes obvious, and almost every "Git disaster" becomes recoverable. This chapter builds
that model.

---

## 1. The problem: tracking changes across time and people

Version control answers three questions:

1. **History**: what did the code look like at any point, and who changed what, when, and
   why?
2. **Parallel work**: how can several people (or several features) change the same code
   at once without trampling each other?
3. **Safety**: how do you experiment freely, knowing you can always get back to a good
   state?

Older centralized systems (Subversion, TFVC) kept history on a server; each developer
had a working copy. Git is **distributed**: every clone contains the *entire* history.
You can commit, branch, view history and diff offline; the server is just another copy
you synchronize with.

---

## 2. The mental model: a content-addressed object store

Git is, at its core, a **key-value database** of objects, where each key is the SHA hash
of the object's content. On top of that it keeps **references** (human-readable names
pointing at objects). That's it.

### Four kinds of objects

| Object | Contains |
|---|---|
| **blob** | The contents of one file (no name, no metadata) |
| **tree** | A directory listing: names, modes and pointers to blobs and other trees |
| **commit** | A pointer to one root tree, pointer(s) to parent commit(s), author, committer, message |
| **tag** (annotated) | A named, signed-able pointer to another object, with a message |

```text
 commit 4f2a…  ("Add triage queue")
 ├─ tree  9c1e…  (project root)
 │   ├─ blob a71b…  README.md
 │   └─ tree 3d0f…  src/
 │       └─ tree …  Beacon.Core/
 │           └─ blob e5c2…  TriageQueue.cs
 └─ parent  b81d…  (previous commit)
```

### Content addressing has big consequences

The ID of every object is a hash of its content:

- **Identical content is stored once.** If 1,000 commits contain the same `README.md`,
  there's one blob.
- **History is tamper-evident.** A commit's ID depends on its tree and its parents' IDs,
  which depend on *their* trees and parents. Change anything in history, and every
  descendant commit gets a new ID. This is why "rewriting history" (rebase, amend) always
  creates *new* commits rather than modifying old ones.
- **Objects are immutable.** Git never changes an object; it only adds new ones.

### A commit is a snapshot, not a diff

A common misconception: commits store *changes*. They don't. Each commit points to a
**complete snapshot** of the project (a tree). Diffs are computed on demand by comparing
two snapshots. (Internally, Git compresses similar objects into *packfiles* using deltas,
but that's a storage optimization invisible to the model.)

### Commits form a graph

Each commit points to its parent(s). The result is a **directed acyclic graph** (DAG) of
history:

```text
 A ◄── B ◄── C ◄── F      (F is a merge commit: two parents, C and E)
        ▲          │
        └── D ◄── E ◄──┘
```

Most commits have one parent. The first commit has none. Merge commits have two (or more).

### References: names for commits

A **branch** is nothing more than a file containing a commit ID: `.git/refs/heads/main`
contains `4f2a…`. That's all. Creating a branch is creating a 41-byte file, which is why
branching in Git is instant.

- **`HEAD`** is a reference to *where you are*: usually it points to a branch
  (`ref: refs/heads/main`), which points to a commit.
- When you commit, Git creates the commit with the current commit as its parent, then
  **moves the branch HEAD points to** forward to the new commit.
- **Remote-tracking branches** (`origin/main`) are your local record of where the
  remote's branches were the last time you fetched.
- **Tags** are references that don't move.

```text
                  HEAD
                   │
                   ▼
                 main
                   │
                   ▼
 A ◄── B ◄── C ◄── D
             ▲
        origin/main   (the remote was at C when we last fetched)
```

> **🧱 Durable:** Branches are movable labels on commits. Almost every Git operation is
> either *creating commits* or *moving labels*. When you're confused, ask "which commits
> exist, and where do the labels point?" and draw it.

### Detached HEAD

If HEAD points directly at a commit instead of at a branch (after `git checkout <sha>` or
during a rebase), you're in **detached HEAD** state. You can look around and even commit,
but no branch moves forward with your commits. Create a branch (`git switch -c rescue`)
before leaving, or your new commits become unreferenced (though still recoverable, as
section 6 explains).

---

## 3. The three areas

Git has three places a file's content can be:

```text
  Working directory  ──git add──►  Index (staging area)  ──git commit──►  Repository (.git)
  (files on disk)                  (the next commit's tree)               (commits, objects)
          ◄────────────────── git restore / git checkout ──────────────────
```

- **Working directory**: the files you edit.
- **Index** (staging area): a proposed snapshot for the next commit. `git add` copies the
  file's current content into the index (as a blob).
- **Repository**: the committed history.

The index exists so you can build a commit deliberately: stage *some* of your changes
(even parts of a file with `git add -p`) and commit them as one logical change, while
keeping others for a separate commit.

`git status` is a report comparing the three: what's different between HEAD and the index
("Changes to be committed"), and between the index and the working directory ("Changes
not staged").

---

## 4. Everyday commands, explained by the model

| Command | What it does to objects and references |
|---|---|
| `git add f` | Writes a blob for `f`'s content; updates the index |
| `git commit` | Writes trees from the index, writes a commit (parent = HEAD), moves the current branch |
| `git switch b` / `git checkout b` | Moves HEAD to branch `b`; updates the index and working directory to match |
| `git branch b` | Creates a reference `b` pointing at the current commit |
| `git fetch` | Downloads objects you don't have; updates `origin/*` remote-tracking branches |
| `git merge x` | Creates a merge commit with two parents (or fast-forwards the branch, if possible) |
| `git pull` | `fetch` + `merge` (or `rebase`, if configured) |
| `git push` | Uploads objects; asks the remote to move its branch to your commit |
| `git reset --soft X` | Moves the current branch to X; leaves index and working directory |
| `git reset --mixed X` | ...and resets the index (the default) |
| `git reset --hard X` | ...and resets the working directory (**discards uncommitted changes**) |
| `git restore f` | Overwrites `f` in the working directory from the index |
| `git revert X` | Creates a **new** commit that undoes X's changes (safe for shared history) |
| `git stash` | Saves uncommitted changes as special commits and cleans the working directory |

### Fast-forward

If you merge a branch that's strictly ahead of yours, Git just moves your label forward;
no merge commit is needed:

```text
 before:  main → C        feature → E   (C ◄── D ◄── E)
 git merge feature (on main)
 after:   main → E        (fast-forward)
```

### Push rejection

`git push` is rejected ("non-fast-forward") when the remote branch has commits you don't
have. The remote would have to *move its label backwards*, losing those commits. The fix
is to fetch, integrate (merge or rebase), and push again. **Force-pushing**
(`--force`) tells the remote to move the label anyway, discarding others' commits. Use
`--force-with-lease`, which refuses if the remote has changed since you last fetched.

---

## 5. Writing good commits

A history is only useful if it can be read. Good commits are:

- **Atomic**: one logical change. "Add triage queue" not "Add triage queue, fix typo,
  update packages, WIP."
- **Buildable**: each commit compiles and passes tests, so tools like `git bisect`
  (section 6) work.
- **Well described**: a short summary line (≈50 characters, imperative mood: "Add",
  "Fix", "Remove"), a blank line, then a body explaining *why* when it isn't obvious.

```text
Add priority queue for ticket triage

Agents asked for a "next ticket" button that always serves the most
urgent and oldest ticket first. A PriorityQueue keyed by (urgency,
createdAt) gives O(log n) enqueue/dequeue. Resolved tickets are skipped
lazily on dequeue rather than removed, because PriorityQueue has no
efficient arbitrary removal.
```

The *what* is in the diff. The *why* exists only in the message, and it's the part future
readers need most.

Many teams use **Conventional Commits** (`feat: add triage queue`, `fix: handle closed
tickets in triage`), which enables automated changelogs and semantic versioning.

### `.gitignore`

Build output (`bin/`, `obj/`), IDE files (`.vs/`, `.idea/`), dependencies
(`node_modules/`), local secrets (`.env`, `appsettings.Local.json`) and OS junk
(`.DS_Store`) should never be committed. `dotnet new gitignore` generates a good .NET one.

> **⚠️ What can go wrong:** Committing a secret (API key, connection string) and then
> deleting it in the next commit **does not remove it**: it's still in history, in every
> clone. Treat any committed secret as compromised: **rotate it immediately**. Then, if
> needed, purge it from history with `git filter-repo` and force-push, knowing that
> existing clones still have it. Prevention: `.gitignore`, secret scanning (GitHub push
> protection), and keeping secrets out of the repo entirely (Book III, Chapter 3).

---

## 6. Recovery: almost nothing is lost

Because objects are immutable and Git only adds, "lost" commits usually still exist;
they're just not referenced by any branch.

### The reflog

Git records every movement of HEAD and branch tips in the **reflog**, locally, for about
90 days by default:

```bash
git reflog
```

```text
8b6dbae HEAD@{0}: reset: moving to HEAD~3
c41e9a0 HEAD@{1}: commit: Add SLA tests
...
```

Did a `reset --hard` throw away three commits? They're at `HEAD@{1}`:

```bash
git branch rescue c41e9a0      # or: git reset --hard HEAD@{1}
```

Bad rebase? `git reflog` shows the commit before the rebase started; reset to it.

> **🔍 Investigation: "I lost my work."** (1) Was it ever committed or stashed? If yes,
> run `git reflog` (or `git fsck --lost-found` for dropped stashes) and find it. (2) If
> it was only staged with `git add`, the blob exists; `git fsck --lost-found` can recover
> file contents. (3) If it was never added, Git never saw it; check your editor's local
> history. Lesson: **commit early and often** on a local branch; you can tidy up later.

### `git bisect`: find the commit that broke something

When something worked last week and doesn't now, `bisect` binary-searches history:

```bash
git bisect start
git bisect bad                 # current commit is broken
git bisect good v1.4.0         # this one worked
# Git checks out a commit halfway; test it, then:
git bisect good   # or: git bisect bad
# ...repeat ~log2(n) times until Git names the first bad commit
git bisect reset
```

With a test script, it's fully automatic: `git bisect run dotnet test --filter SlaRules`.
Across 1,000 commits, that's about 10 steps. This is one of the strongest arguments for
small, buildable commits.

### Inspecting history

```bash
git log --oneline --graph --all          # the commit graph
git log -p -- src/Beacon.Core/Tickets/Ticket.cs   # history of one file with diffs
git log -S "IsBreaching"                 # commits that added or removed this string
git blame -w src/.../SlaRules.cs         # who last changed each line (and in which commit)
git show <sha>                           # one commit
git diff main...feature                  # what feature changed since it diverged from main
```

---

## 7. In practice: putting Beacon under version control

```bash
cd beacon
dotnet new gitignore
git init -b main
git add .
git status            # check: no bin/, obj/ or secrets staged
git commit -m "Initial Beacon solution: core domain, CLI and tests"
```

Look at what you created:

```bash
git cat-file -p HEAD             # the commit: tree, author, message
git cat-file -p HEAD^{tree}      # the root tree: README, src, tests...
git ls-tree -r HEAD | head       # every blob in the snapshot
ls .git/refs/heads               # main: a file containing the commit ID
cat .git/HEAD                    # ref: refs/heads/main
```

Then publish it to GitHub (create an empty repository first):

```bash
git remote add origin https://github.com/<you>/beacon.git
git push -u origin main          # -u: set origin/main as main's upstream
```

From now on, each chapter's changes to Beacon can be a commit, or a branch and a pull
request, which is the workflow of the next two chapters.

---

## 8. What can go wrong

- **Committing secrets** (section 5).
- **Huge binary files** bloating the repository forever. Use Git LFS for large assets, or
  keep them out of Git.
- **`reset --hard` with uncommitted work**: the one truly destructive operation. Uncommitted
  changes that were never staged are gone.
- **Force-pushing shared branches**, rewriting history others have built on.
- **Line-ending chaos** between Windows and Linux. Set `* text=auto` in `.gitattributes`.
- **Giant "WIP" commits** that make history, review and bisect useless.
- **Working for days without committing or pushing**: one disk failure from losing it all.

---

## 9. How an experienced engineer thinks about this

- **Think in graphs and labels.** Draw the commits and where branches point; then choose
  the command that produces the picture you want.
- **Commit locally often; publish deliberately.** Local history can be reshaped freely;
  shared history shouldn't be rewritten.
- **History is documentation.** Write commit messages for the developer who runs
  `git blame` on this line in two years.
- **Almost everything is recoverable.** Reach for `reflog` before panicking.

---

## 10. Check yourself

**Questions**

1. What are Git's four object types? What does a commit point to?
2. Why does changing an old commit change the IDs of all later commits?
3. What is a branch, physically? What is HEAD?
4. What's the difference between the working directory, the index and the repository?
5. What's the difference between `reset --soft`, `--mixed` and `--hard`?
6. Why is `revert` safer than `reset` on a shared branch?
7. You ran `git reset --hard HEAD~2` by mistake. How do you get the commits back?

**Exercises**

1. In a scratch repository, use `git cat-file -p` to walk from a commit to its tree to a
   blob, and read the file's content.
2. Create two identical files in different folders, commit, and verify with `git ls-tree -r`
   that they share one blob ID.
3. Make three commits, `reset --hard` back two, and recover them with the reflog.
4. Use `git bisect run` to find a deliberately introduced failing test in a sequence of
   ten commits.

**Interview-style questions**

- "Explain the difference between `git merge` and `git rebase`." (Next chapter.)
- "What's in a Git commit?"
- "How would you recover a commit you accidentally deleted?"
- "Someone committed an API key. What do you do?"

---

## 11. Going deeper

- [*Pro Git*](https://git-scm.com/book) by Scott Chacon and Ben Straub (free) — especially
  chapter 10, "Git Internals."
- [Git documentation](https://git-scm.com/docs)
- [Oh Shit, Git!?!](https://ohshitgit.com/) — short recipes for common recoveries.

**Next:** [Chapter 2 — Branching, Merging and Rebasing](02-branching-merging-and-rebasing.md)
uses this model to work in parallel and bring changes back together.
