# Writing guide for this book

This repo is an mdBook: *The Modern Software Engineer*. `src/front/master-plan.md` is the
authoritative spec; read it before writing. Chapter 1 of Book I is the reference for voice
and depth.

## Conventions

- One file per chapter under `src/NN-book/`. Replace the "Status: planned" placeholder
  completely; keep the H1 title identical to `SUMMARY.md`.
- Follow the chapter template: problem → mental model → in practice (Beacon) → what can go
  wrong → when not to use it → how an experienced engineer thinks → check yourself
  (questions, exercises, interview-style questions) → going deeper → "Next:" link.
- Callouts: `> **🧱 Durable:**`, `> **🔄 Current (as of <Month Year>):**`,
  `> **⚠️ What can go wrong:**`, `> **🧭 When not to use it:**`, `> **🔍 Investigation:**`.
  Version-specific facts go only in 🔄 callouts.
- Target 4,000–7,000 words. Plain, direct prose; explain *why*; compare with C#/.NET when
  teaching other languages.
- Current baseline (October 2026): .NET 10 / C# 14 (LTS), PostgreSQL 18, TypeScript 5.x,
  React 19, Node 24 LTS, Python 3.14.

## Running project: Beacon

A team knowledge base + support desk. Solution `Beacon.sln` with `src/Beacon.Core`
(domain), `src/Beacon.Cli`, later `src/Beacon.Api` (ASP.NET Core), `src/Beacon.Infrastructure`
(EF Core/PostgreSQL), `tests/Beacon.Core.Tests` (xUnit), `web/` (React + TS + Vite).
Domain so far: `Beacon.Core.Tickets.Ticket` (sealed record: Id, Title, Status),
`TicketStatus { Open, InProgress, Resolved, Closed }`. Grow it consistently; later
entities: `Article` (knowledge base), `Comment`, `User`, `Team`.

## Publishing

After each chapter: update the book's `README.md` status line, add a line to
`src/front/changelog.md`, run `mdbook build`, commit, push to `main` (CI deploys to
GitHub Pages). Updates to *existing* content after review go through pull requests.
