# Relational Thinking

> **🔄 Current (as of October 2026):** Examples target PostgreSQL 18. Almost everything in
> this chapter applies to any relational database (SQL Server, MySQL, Oracle, SQLite).

Most application bugs eventually become data bugs. A missing constraint lets two users
claim the same username. A poorly chosen key makes a merge of two systems impossible. A
denormalized column drifts out of sync with its source, and nobody notices for months.
Code can be redeployed in minutes; bad data, once written, has to be found and repaired,
sometimes by hand.

This chapter is about designing data so it stays correct: the relational model, keys and
constraints, normalization, and how to turn a domain like Beacon's into a schema.

---

## 1. The problem: data outlives code

Applications get rewritten; databases persist. Beacon's ticket data may outlive its C#
code, its API design and its frontend framework. Other systems (reporting, analytics,
integrations) will read it directly. So the database must protect its own correctness,
independently of any particular application.

A relational database does this with a **schema**: a precise description of what data
exists, how it relates, and which rules it must always satisfy, enforced by the database
itself.

---

## 2. The mental model: relations, tuples and attributes

E.F. Codd's **relational model** (1970) is the foundation of SQL databases:

| Theory | SQL | Meaning |
|---|---|---|
| Relation | Table | A set of facts of the same shape |
| Tuple | Row | One fact |
| Attribute | Column | One property, with a type (domain) |

Key ideas that shape everything:

- **A table is a set of facts.** Each row asserts something true: "Ticket 42, titled
  'VPN drops', has priority High."
- **Rows have no inherent order.** Without `ORDER BY`, the database may return them in any
  order, and the order can change between executions.
- **Rows are identified by values (keys)**, not by position or memory address.
- **Relationships are expressed by matching values** (a `ticket_id` column in `comments`),
  not by pointers. Any relationship can be queried in either direction.
- **Operations work on whole sets.** "Close all tickets older than 90 days" is one
  statement, not a loop.

> **🧱 Durable:** Think in **sets**, not loops. The most common mistake application
> developers make in SQL is fetching rows one at a time and processing them in code, when
> one set-based statement would do it faster and atomically.

---

## 3. Keys

### Primary keys

Every table needs a **primary key**: a column (or columns) whose value uniquely identifies
each row, never null, and ideally never changing.

**Natural keys** come from the domain (an email address, an ISO country code).
**Surrogate keys** are generated values with no business meaning (an auto-incrementing
number, a UUID).

| | Natural key | Surrogate key |
|---|---|---|
| Example | `email`, `country_code` | `id bigint generated always as identity`, `id uuid` |
| Meaningful | Yes | No |
| Can change? | Often (people change emails) | Never |
| Size | Varies | Small, fixed |

Use **surrogate primary keys** for entities that can change (users, tickets), and add a
**unique constraint** on natural keys that must be unique (`email`). Natural keys work well
for stable reference data (`country_code char(2)`).

### Integer vs UUID

| | `bigint identity` | `uuid` |
|---|---|---|
| Size | 8 bytes | 16 bytes |
| Generated | By the database, sequentially | Anywhere (app, client, database) |
| Index locality | Excellent (always appended) | Random v4 UUIDs scatter inserts across the index |
| Guessable / leaks volume | Yes (ticket 1042 implies ~1041 before it) | No |
| Merging data across databases | Conflicts | No conflicts |

**UUIDv7** (time-ordered UUIDs) combines the benefits: globally unique, generated anywhere,
and roughly sequential, so index locality is good. .NET 9+ has `Guid.CreateVersion7()`,
and PostgreSQL 18 adds a built-in `uuidv7()` function.

Beacon uses `bigint` identity keys internally, exposed as `"T-42"` (Book III treated IDs
as opaque strings, so this can change later). Guessable IDs aren't a security problem
when every access is authorized (Book III, Chapter 6), though they do reveal volume.

### Foreign keys

A **foreign key** says "this column's value must exist as a key in that table":

```sql
create table comments (
    id         bigint generated always as identity primary key,
    ticket_id  bigint not null references tickets (id) on delete cascade,
    ...
);
```

The database now *guarantees* no comment points to a nonexistent ticket. `on delete`
decides what happens when the parent is deleted: `restrict`/`no action` (refuse),
`cascade` (delete children too), `set null`.

> **⚠️ What can go wrong:** Skipping foreign keys "for performance" or "because the app
> checks" leads to orphaned rows, which every query and report then has to work around.
> The app *doesn't* always check: scripts, migrations, other services and bugs bypass it.
> Keep foreign keys, and index the referencing columns (Chapter 4).

---

## 4. Constraints: rules the database enforces

Constraints are the database's version of invariants (Book I, Chapter 3), enforced for
every writer:

```sql
create table tickets (
    id           bigint generated always as identity primary key,
    title        text not null check (length(title) between 1 and 200),
    priority     smallint not null check (priority between 0 and 3),
    status       smallint not null default 0 check (status between 0 and 3),
    team_id      bigint not null references teams (id),
    reporter_id  text not null,
    assignee_id  text null,
    created_at   timestamptz not null default now(),
    resolved_at  timestamptz null,
    version      integer not null default 1,

    constraint resolved_has_time check (
        (status in (2, 3)) = (resolved_at is not null)   -- resolved/closed ⇔ resolved_at set
    )
);
```

| Constraint | Guarantees |
|---|---|
| `not null` | A value is always present |
| `unique` | No duplicates (in a column or combination) |
| `primary key` | `unique` + `not null` |
| `foreign key` | References exist |
| `check` | An arbitrary condition on the row |
| `exclude` (PostgreSQL) | No two rows "overlap" (e.g. booking time ranges) |

Constraints are your last line of defense. The application should still validate (for good
error messages), but the database guarantees correctness even when the application has a
bug, a script runs, or a race condition slips past application checks.

### Race conditions and uniqueness

"Check if the username exists, then insert" in application code is a race (Book I,
Chapter 10): two requests can both check, both see nothing, and both insert. Only a
**unique constraint** reliably prevents duplicates. The application catches the
constraint violation and returns a friendly `409`.

---

## 5. Normalization

**Normalization** organizes data so each fact is stored **once**. Its purpose is to
prevent **update anomalies**: inconsistencies that arise when the same fact is stored in
several places.

Consider a denormalized design:

```text
tickets
id | title          | team_name | team_lead_email     | assignee_name | assignee_email
42 | VPN drops      | Network   | lead@example.com    | Maria Lopez   | maria@example.com
43 | Printer offline| Network   | lead@example.com    | Omar Haddad   | omar@example.com
```

Problems:

- **Update anomaly**: the Network team's lead changes. You must update every ticket row; miss
  one and the data contradicts itself.
- **Insert anomaly**: you can't record a new team until it has a ticket.
- **Delete anomaly**: delete the last Network ticket and you lose the team's lead email.

### The normal forms, practically

- **1NF**: each column holds one atomic value; no repeating groups (`tag1, tag2, tag3`
  columns, or comma-separated lists in a column).
- **2NF**: every non-key column depends on the *whole* key (matters for composite keys).
- **3NF**: every non-key column depends on the key, *the whole key, and nothing but the
  key*. `team_lead_email` depends on the team, not the ticket, so it belongs in `teams`.

Normalized:

```text
teams (id, name, lead_user_id)
users (id, name, email, team_id)
tickets (id, title, team_id, assignee_id, ...)
```

Each fact (a team's lead, a user's email) lives in exactly one row.

> **🧱 Durable:** "One fact, one place" is the whole idea. Normal forms are just a formal
> way to find facts stored in the wrong place.

### Denormalization: deliberately breaking the rule

Sometimes you *want* redundancy, for read performance:

- A `comment_count` column on `tickets` instead of counting comments on every list query.
- A snapshot of a price on an order line (the product's price changes; the order's
  shouldn't). This one is not really denormalization: it's a *different fact* ("the price
  at the time of the order").
- Read models and search indexes built for specific queries (Book XIII, CQRS).

Denormalize **deliberately**, with a mechanism that keeps the copy consistent (same
transaction, a trigger, or an event with known lag), and only after measuring a need.

---

## 6. Modeling relationships

| Relationship | Example | How |
|---|---|---|
| One-to-many | A ticket has many comments | Foreign key on the "many" side: `comments.ticket_id` |
| Many-to-many | Tickets have many tags; tags apply to many tickets | A **junction table**: `ticket_tags (ticket_id, tag_id)` with a composite primary key |
| One-to-one | A user has one profile | Foreign key with a unique constraint (or same primary key) |
| Optional relationship | A ticket may have an assignee | Nullable foreign key |
| Hierarchy | Article categories with subcategories | Self-referencing foreign key (`parent_id`), queried with recursive CTEs (Chapter 2) |

### Polymorphic associations: a trap

"A comment can belong to a ticket *or* an article" is often modeled as
`(commentable_type text, commentable_id bigint)`. The database can't enforce a foreign key
on that, and every query needs a type filter. Alternatives: separate tables
(`ticket_comments`, `article_comments`), or two nullable foreign keys with a check that
exactly one is set.

---

## 7. Choosing data types

Types are constraints too. Choose the most precise type:

| Data | PostgreSQL type | Avoid |
|---|---|---|
| Identifiers | `bigint` (identity) or `uuid` | `int` for tables that might grow large (2.1 billion limit) |
| Text | `text` (+ `check` on length) or `varchar(n)` | `char(n)` (pads with spaces) |
| Money | `numeric(12, 2)` or integer minor units | `float`/`real`/`double precision` (rounding errors) |
| Points in time | `timestamptz` | `timestamp` without time zone for instants |
| Calendar dates | `date` | Strings |
| True/false | `boolean` | `int`, `'Y'/'N'` |
| Enumerations | `smallint` + `check`, a lookup table, or a PostgreSQL `enum` type | Free text |
| Semi-structured data | `jsonb` (Chapter 3) | `text` containing JSON |

PostgreSQL's `timestamptz` stores an absolute instant (internally in UTC) and converts on
display, which matches .NET's `DateTimeOffset`. `timestamp` (without time zone) stores a
"wall clock" reading with no zone; it's correct only for things like "store opens at 09:00
local time."

---

## 8. In practice: Beacon's schema

Here's Beacon's core schema, designed from the domain built so far. It goes in a migration
(Chapter 7 generates it with EF Core; writing it by hand first makes the design explicit).

```sql
-- Teams and users (users come from the identity provider; we store what we need)
create table teams (
    id    bigint generated always as identity primary key,
    name  text not null unique check (length(name) between 1 and 100)
);

create table users (
    id            text primary key,                 -- the identity provider's 'sub'
    display_name  text not null,
    email         text not null unique,
    team_id       bigint null references teams (id),
    role          text not null check (role in ('customer', 'agent', 'lead'))
);

-- Tickets
create table tickets (
    id           bigint generated always as identity primary key,
    title        text not null check (length(title) between 1 and 200),
    description  text null check (length(description) <= 10000),
    priority     smallint not null check (priority between 0 and 3),   -- Low..Urgent
    status       smallint not null default 0 check (status between 0 and 3),
    team_id      bigint not null references teams (id),
    reporter_id  text not null references users (id),
    assignee_id  text null references users (id),
    created_at   timestamptz not null default now(),
    resolved_at  timestamptz null,
    version      integer not null default 1,
    constraint resolved_has_time check ((status in (2, 3)) = (resolved_at is not null))
);

create table comments (
    id          bigint generated always as identity primary key,
    ticket_id   bigint not null references tickets (id) on delete cascade,
    author_id   text not null references users (id),
    body        text not null check (length(body) between 1 and 5000),
    created_at  timestamptz not null default now()
);

-- Tags: many-to-many
create table tags (
    id    bigint generated always as identity primary key,
    name  text not null unique
);

create table ticket_tags (
    ticket_id  bigint not null references tickets (id) on delete cascade,
    tag_id     bigint not null references tags (id) on delete cascade,
    primary key (ticket_id, tag_id)
);

-- Knowledge base
create table articles (
    id            bigint generated always as identity primary key,
    slug          text not null unique check (slug ~ '^[a-z0-9-]+$'),
    title         text not null,
    body          text not null,
    status        text not null default 'draft' check (status in ('draft', 'published', 'archived')),
    published_at  timestamptz null,
    updated_at    timestamptz not null default now()
);
```

Design decisions worth noting:

- **`users.id` is the identity provider's `sub`** (a natural key from another system that
  never changes for a given user). Display name and email live here because the API needs
  them, but the identity provider remains the source of truth for authentication.
- **Enums stored as `smallint` with `check` constraints**, matching the C# enum values.
  Lookup tables would add joins; PostgreSQL `enum` types are harder to evolve. Any of the
  three is defensible; the important thing is the constraint.
- **`resolved_has_time` check** encodes a domain invariant in the database.
- **`on delete cascade`** for comments and tags (they have no meaning without their
  ticket), but **not** for `team_id` or `reporter_id` (deleting a team that has tickets
  should be refused).
- **No `comment_count` column yet.** We'll add denormalization only if measurement shows a
  need.

Run it against a local PostgreSQL (Book VIII covers Docker in depth; for now):

```bash
docker run -d --name beacon-db -e POSTGRES_PASSWORD=dev -e POSTGRES_USER=beacon \
  -e POSTGRES_DB=beacon -p 5432:5432 postgres:18
psql "postgresql://beacon:dev@localhost:5432/beacon" -f schema.sql
```

Then try to break it:

```sql
insert into tickets (title, priority, team_id, reporter_id) values ('', 1, 1, 'u1');
-- ERROR: new row violates check constraint "tickets_title_check"

insert into comments (ticket_id, author_id, body) values (99999, 'u1', 'hello');
-- ERROR: insert or update violates foreign key constraint "comments_ticket_id_fkey"

update tickets set status = 2 where id = 1;   -- resolved without resolved_at
-- ERROR: new row violates check constraint "resolved_has_time"
```

Every one of these errors is a bug that can't reach production data.

---

## 9. What can go wrong

- **No constraints**, relying on the application: orphaned rows, duplicates, invalid states.
- **Comma-separated values** in a column ("tags": "vpn,network"): unqueryable, unindexable,
  no integrity.
- **Floating point for money.**
- **`timestamp` without time zone for instants**, causing off-by-hours bugs around
  time-zone and DST changes.
- **Natural keys that change** (emails, usernames) used as primary keys and foreign keys.
- **Premature denormalization** with no mechanism to keep copies in sync.
- **Entity-attribute-value designs** (`attributes(entity_id, name, value)`) to avoid schema
  changes. Everything becomes text, constraints disappear, and queries become unreadable.
  Use `jsonb` for genuinely flexible data instead (Chapter 3).
- **`int` keys overflowing** at 2.1 billion on high-volume tables.

---

## 10. How an experienced engineer thinks about this

- **Model facts, not screens.** The UI will change; the facts won't.
- **Let the database enforce invariants.** Constraints are cheap; corrupt data is expensive.
- **Normalize by default; denormalize deliberately**, with a consistency mechanism.
- **Choose types precisely.** Every type is a constraint.
- **Design keys for the long term**: stable, small, unique, and not leaking meaning you
  might regret.

---

## 11. Check yourself

**Questions**

1. Why is it important that rows have no inherent order?
2. Compare natural and surrogate keys. When would you use each?
3. What are the trade-offs between `bigint` identity keys and UUIDs? What does UUIDv7 change?
4. What update, insert and delete anomalies does normalization prevent? Give examples.
5. Why must uniqueness be enforced by a constraint rather than an application check?
6. Why use `numeric` rather than `double precision` for money?
7. What's wrong with polymorphic associations?

**Exercises**

1. Normalize this table to 3NF: `orders(order_id, customer_name, customer_email,
   product_name, product_price, quantity, order_date)`.
2. Add a `ticket_watchers` many-to-many table (users watching tickets) to Beacon's schema.
3. Add a constraint that an article's `published_at` is set exactly when status is
   `published`.
4. Model "an agent's working hours per weekday" and decide between `timestamptz`,
   `timestamp` and `time`.

**Interview-style questions**

- "Design a database schema for a support ticket system."
- "What is normalization? When would you denormalize?"
- "Natural or surrogate keys? Integer or UUID?"

---

## 12. Going deeper

- [PostgreSQL documentation: Data Definition](https://www.postgresql.org/docs/current/ddl.html)
- C.J. Date, *Database Design and Relational Theory* — rigorous foundations.
- Bill Karwin, *SQL Antipatterns* — the common mistakes, with fixes.

**Next:** [Chapter 2 — SQL in Depth](02-sql-in-depth.md) covers querying this schema:
joins, aggregation, CTEs and window functions.
