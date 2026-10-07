# The Master Plan

This page is the book's constitution. Every chapter is written against it, so that
chapter 80 reads like it belongs to the same book as chapter 1.

## Goal

Take a developer with C#/.NET experience and build them into a modern full-stack,
cloud and AI-capable software engineer, with the judgment to apply that knowledge in
real projects and interviews.

The book does **not** promise "everything" about each technology. It covers what a
professional full-stack engineer on this stack needs to know, understand and apply,
with enough depth to know where to go deeper.

## Reader

- Has written C# professionally or semi-professionally.
- Wants to understand *why*, not just *how*.
- Reads the book over months and keeps returning to it as it is updated.

*Open question: the exact starting level in C# (async/await, generics, LINQ) sets the
pace of Book I.*

## Principles

1. **Why before how.** Every technology starts with the problem it solves.
2. **Costs, not just benefits.** Every tool or pattern gets a "when not to use it" section.
3. **One system.** Examples build the [running project](running-project.md).
4. **Durable vs. current.** Concepts go in the main text; version-specific details go in
   🔄 callouts that carry a date and are reviewed regularly.
5. **Investigation skills.** Each book includes at least one 🔍 troubleshooting walkthrough
   (a slow query, a failing API, a broken container, a misbehaving Linux service).
6. **Compare with what you know.** New languages and runtimes are explained in contrast
   with C#/.NET.

## Chapter template

Each chapter follows roughly this shape:

1. **The problem** — why this exists.
2. **The mental model** — how it works underneath.
3. **In practice** — building it into the running project.
4. **What can go wrong** — failure modes and traps.
5. **When not to use it** — costs and alternatives.
6. **How an experienced engineer thinks about it** — judgment and trade-offs.
7. **Check yourself** — questions and exercises, including interview-style questions.
8. **Going deeper** — where to look next.

Target length: 5,000–9,000 words per chapter.

## Structure

| Book | Topic |
|---|---|
| I | Programming & C# |
| II | Git & Developer Workflow |
| III | .NET & Backend |
| IV | SQL & PostgreSQL |
| V | JavaScript & TypeScript |
| VI | React & Frontend |
| VII | Full-Stack Architecture |
| VIII | Docker & Linux |
| IX | Azure, Cloud & DevOps |
| X | Python |
| XI | AI Application Engineering |
| XII | Rust |
| XIII | Architecture, Security & System Design |
| XIV | Real-World Projects |

Changes from the original 13-book outline: Git moved to Book II because it is used from
the first project onward; CI/CD joined the Azure book; Python now comes before AI because
much AI tooling assumes it. The full chapter list is in the sidebar.

## How the book stays current

- A scheduled review checks release notes and documentation for the fast-moving areas
  (.NET, ASP.NET Core, EF Core, PostgreSQL, TypeScript, React, Docker, Azure, AI APIs and SDKs).
- Proposed edits arrive as a **pull request** with a summary of what changed and why.
- Nothing is merged without review.
- Each merged update is logged on [What's Changed](changelog.md).
