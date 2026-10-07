# PostgreSQL Specifics

> **🔄 Current (as of October 2026):** PostgreSQL 18 is the current major version
> (released September 2025), adding asynchronous I/O for faster reads, a built-in
> `uuidv7()` function, virtual generated columns, and OAuth authentication support.
> PostgreSQL ships a new major version every year, each supported for five years.

PostgreSQL has become the default relational database for new projects across most of the
industry: open source, standards-compliant, extremely reliable, and extensible enough to
replace several specialized systems. Managed versions exist on every cloud (Azure Database
for PostgreSQL, Amazon RDS/Aurora, Google Cloud SQL).

Chapter 2 was portable SQL. This chapter covers what's specific to PostgreSQL: its
architecture, its rich type system (especially `jsonb` and arrays), full-text search,
extensions, roles and schemas, and the operational features you'll rely on.

---

## 1. The problem: one database, many workloads

A typical application needs more than tables of rows:

- semi-structured data (settings, integration payloads, custom fields),
- text search,
- geospatial queries,
- vector similarity search for AI (Book XI),
- queues and scheduled jobs,
- time series.

Each could be a separate specialized system: a document store, a search engine, a vector
database. Every additional system adds operations, consistency problems between stores,
and cost. PostgreSQL can often handle these workloads well enough to avoid extra systems,
at least until scale or requirements prove otherwise.

> **🧱 Durable:** "Use the boring technology you already have until it demonstrably isn't
> enough" is one of the most reliable architecture principles. A second datastore is a
> second thing to back up, secure, monitor, upgrade and keep consistent.

---

## 2. The mental model: how PostgreSQL works

```text
 Clients ──► postmaster ──► one backend PROCESS per connection
                            │
                            ├─ Shared buffers (cache of 8 KB data pages)
                            ├─ WAL (write-ahead log): every change is logged first
                            └─ Background workers: checkpointer, autovacuum, WAL writer, replication
                 Data files on disk (tables and indexes as 8 KB pages)
```

Key consequences:

- **One process per connection.** Connections are relatively expensive (memory, process
  creation). Hundreds are fine; thousands need a **connection pooler** (PgBouncer, or
  built-in pooling in managed services), and your application's pool size matters
  (Chapter 8).
- **Write-ahead logging.** Changes are written to the WAL before the data files, so a crash
  can be recovered by replaying the log. The WAL also powers replication and point-in-time
  recovery (Chapter 8).
- **MVCC (Multi-Version Concurrency Control).** An `update` doesn't overwrite a row; it
  writes a *new version* and marks the old one as obsolete. Readers see a consistent
  snapshot without blocking writers. Chapter 5 explains the consequences; one of them is
  that old row versions must be cleaned up by **VACUUM**.

### VACUUM and autovacuum

Every update and delete leaves behind dead row versions. **Autovacuum** runs in the
background to:

- reclaim space from dead rows,
- update planner statistics (`analyze`),
- prevent *transaction ID wraparound*, a rare but serious condition in very busy databases.

You rarely run VACUUM manually, but you should know its symptoms: tables that grow
("bloat") despite stable row counts, or queries slowing as a table accumulates dead rows,
often because a long-running transaction prevents cleanup. Monitor `pg_stat_user_tables`
(`n_dead_tup`, `last_autovacuum`).

---

## 3. Data types beyond the basics

### `jsonb`

PostgreSQL stores JSON in a decomposed binary format (`jsonb`), queryable and indexable:

```sql
alter table tickets add column metadata jsonb not null default '{}';

update tickets
set metadata = '{"source": "email", "customer": {"plan": "enterprise", "region": "eu"}}'
where id = 42;

select id, title
from tickets
where metadata ->> 'source' = 'email'                      -- ->> extracts as text
  and metadata -> 'customer' ->> 'plan' = 'enterprise';    -- -> extracts as jsonb

select * from tickets where metadata @> '{"customer": {"region": "eu"}}';   -- containment
```

With a **GIN index** (Chapter 4), containment queries (`@>`) are fast even on large tables.
PostgreSQL also supports the SQL/JSON standard functions (`json_table`, `json_exists`,
`json_value`) for more complex extraction.

> **🧭 When not to use `jsonb`:** For data you filter, join, constrain or report on
> regularly, use proper columns. JSON loses type checking, foreign keys and most
> constraints, and queries become harder to read and optimize. `jsonb` shines for
> genuinely variable data: integration payloads, per-customer custom fields, settings,
> and audit "before/after" snapshots. A hybrid (core fields as columns, extras in `jsonb`)
> is often ideal.

### Arrays

```sql
alter table articles add column keywords text[] not null default '{}';
select * from articles where 'vpn' = any(keywords);
select * from articles where keywords && array['vpn', 'wifi'];   -- overlap
```

Arrays are convenient for small lists of simple values that are always read with the row.
For anything you query relationally (tags you want to count, rename, or join), use a
proper junction table (Chapter 1).

### Other useful types

| Type | Use |
|---|---|
| `timestamptz`, `interval`, `date` | Time (Chapter 1), with rich arithmetic: `now() - interval '2 days'` |
| `tstzrange`, `daterange` | Ranges: "on-call from 09:00 to 17:00," with overlap operators and exclusion constraints |
| `uuid` | Identifiers; `uuidv7()` in PostgreSQL 18 |
| `citext` (extension) | Case-insensitive text (emails, usernames) |
| `inet`, `cidr` | IP addresses and networks |
| `tsvector`, `tsquery` | Full-text search (section 4) |
| `vector` (pgvector extension) | Embeddings for similarity search (Book XI) |

### Generated columns

```sql
alter table tickets add column is_open boolean
    generated always as (status in (0, 1)) stored;
```

A generated column is computed from other columns and kept in sync automatically: a safe
form of denormalization. PostgreSQL 18 also supports `virtual` generated columns (computed
on read, not stored) and makes them the default.

### Exclusion constraints

Prevent overlapping on-call shifts for the same agent:

```sql
create extension if not exists btree_gist;

create table on_call_shifts (
    user_id  text not null references users (id),
    during   tstzrange not null,
    exclude using gist (user_id with =, during with &&)
);
```

Try to insert overlapping shifts for the same user, and the database refuses. This is an
invariant that's genuinely hard to enforce in application code without race conditions.

---

## 4. Full-text search

`like '%vpn%'` can't use normal indexes, doesn't understand language (searching "connect"
won't find "connecting"), and has no relevance ranking. PostgreSQL's full-text search does
all three:

```sql
alter table articles add column search tsvector
    generated always as (
        setweight(to_tsvector('english', coalesce(title, '')), 'A') ||
        setweight(to_tsvector('english', coalesce(body, '')), 'B')
    ) stored;

create index articles_search_idx on articles using gin (search);

select slug, title, ts_rank(search, q) as rank
from articles, websearch_to_tsquery('english', 'vpn not connecting') q
where search @@ q and status = 'published'
order by rank desc
limit 10;
```

- `to_tsvector` normalizes text into **lexemes** (stemmed words without stop words):
  "connecting" → `connect`.
- `websearch_to_tsquery` parses user-friendly syntax (quotes, `or`, `-exclude`).
- **Weights** rank title matches above body matches.
- The GIN index makes it fast.

For Beacon's knowledge base, this is likely enough for a long time. Dedicated search
engines (Elasticsearch, OpenSearch, Azure AI Search) become worthwhile for typo tolerance,
faceting at scale, multiple languages, or very large corpora. Book XI adds **semantic**
search with embeddings, and combining it with full-text search (*hybrid search*) is
usually better than either alone.

---

## 5. Extensions

PostgreSQL's extension system adds types, functions, index methods and more:

| Extension | Adds |
|---|---|
| `pg_stat_statements` | Statistics on every query: calls, total time, rows. **Enable it everywhere** (Chapter 6). |
| `pgcrypto` | Cryptographic functions |
| `citext` | Case-insensitive text type |
| `pg_trgm` | Trigram similarity: fuzzy search, fast `like '%x%'` with GIN indexes |
| `btree_gist` | Needed for exclusion constraints mixing equality and ranges |
| `postgis` | Geospatial types and queries: the industry standard |
| `pgvector` | Vector similarity search for embeddings (Book XI) |
| `pg_cron` | Scheduled jobs inside the database |
| `timescaledb` | Time-series optimization (where available) |

```sql
create extension if not exists pg_stat_statements;
create extension if not exists pg_trgm;
```

Managed cloud services support a curated list of extensions; check before designing around
one.

---

## 6. Roles, schemas and privileges

### Roles

PostgreSQL has **roles**, which can log in (users) or group privileges. Follow least
privilege (Chapter 8 goes deeper):

```sql
create role beacon_migrator login password '...';    -- owns the schema; used only by migrations
create role beacon_app login password '...';         -- the running application
create role beacon_readonly login password '...';    -- reporting

grant usage on schema public to beacon_app, beacon_readonly;
grant select, insert, update, delete on all tables in schema public to beacon_app;
grant select on all tables in schema public to beacon_readonly;
alter default privileges for role beacon_migrator in schema public
    grant select, insert, update, delete on tables to beacon_app;
```

The application can read and write data but can't drop tables or alter the schema, so a
SQL injection or a compromised app server can do less damage.

### Schemas

A **schema** is a namespace within a database: `public.tickets`, `reporting.daily_stats`,
`audit.ticket_changes`. Use schemas to separate modules (Book XIII, modular monoliths),
to apply privileges per area, or to keep extensions and application tables apart. The
`search_path` setting decides which schemas unqualified names resolve against.

### Row-level security

PostgreSQL can enforce per-row access rules inside the database:

```sql
alter table tickets enable row level security;

create policy team_isolation on tickets
    using (team_id = current_setting('app.team_id')::bigint);
```

The application sets `app.team_id` per connection or transaction, and every query is
automatically filtered. RLS is powerful for **multi-tenant** systems as a second line of
defense behind application authorization (Book III, Chapter 6). It requires care with
connection pooling (the setting must be set per transaction) and is bypassed by table
owners and superusers.

---

## 7. Useful operational features

- **`explain analyze`**: how a query actually executed (Chapter 6).
- **`pg_stat_activity`**: what every connection is doing right now, including long-running
  queries and idle-in-transaction sessions.
- **`pg_locks`**: who's waiting on whom (Chapter 5).
- **`statement_timeout`** and **`idle_in_transaction_session_timeout`**: kill runaway
  queries and abandoned transactions. Set them for the application role:

```sql
alter role beacon_app set statement_timeout = '15s';
alter role beacon_app set idle_in_transaction_session_timeout = '60s';
```

- **`listen` / `notify`**: lightweight pub/sub between sessions. Useful for "something
  changed" signals (e.g. cache invalidation), not for durable messaging.
- **`copy`**: bulk import and export, orders of magnitude faster than individual inserts.
- **Logical replication** and **read replicas** for scaling reads and migrations (Chapter 8).

---

## 8. In practice: PostgreSQL features in Beacon

### Knowledge-base search

Add the generated `search` column and GIN index from section 4 to `articles`. The API gets
a search endpoint (Chapter 7 shows the EF Core side):

```sql
-- the query the endpoint runs, parameterized
select id, slug, title,
       ts_headline('english', body, q, 'MaxWords=30, MinWords=10') as snippet,
       ts_rank(search, q) as rank
from articles, websearch_to_tsquery('english', @query) q
where status = 'published' and search @@ q
order by rank desc
limit 20;
```

`ts_headline` returns a snippet with matching terms highlighted, perfect for search
results.

### Fuzzy matching for tags

Agents mistype tag names ("netwrok"). Trigram similarity suggests the right one:

```sql
create extension if not exists pg_trgm;
create index tags_name_trgm on tags using gin (name gin_trgm_ops);

select name, similarity(name, 'netwrok') as score
from tags
where name % 'netwrok'            -- similarity above threshold (default 0.3)
order by score desc
limit 5;
```

### Custom fields with `jsonb`

Enterprise customers want custom fields on tickets ("Contract number," "Affected site").
These vary per customer, so `jsonb` fits:

```sql
alter table tickets add column custom_fields jsonb not null default '{}';
create index tickets_custom_fields_idx on tickets using gin (custom_fields jsonb_path_ops);

-- find tickets for contract C-1001
select id, title from tickets where custom_fields @> '{"contractNumber": "C-1001"}';
```

The *definitions* of custom fields (name, type, required) live in a normal table,
`custom_field_definitions`, so the API can validate values on the way in (Book III,
Chapter 5). Flexible storage, but not unvalidated.

### Database roles

Beacon gets the three roles from section 6. Migrations (Chapter 7) run as
`beacon_migrator`; the API connects as `beacon_app` with statement and idle timeouts;
the reporting endpoint uses a separate connection string with `beacon_readonly`, so a
bug in reporting code can't modify data.

---

## 9. What can go wrong

- **Too many connections**: one process each; exhaust memory or `max_connections`. Use
  pooling.
- **Long-running transactions** blocking vacuum, causing bloat and lock contention.
- **`jsonb` for everything**, losing constraints and making queries opaque.
- **`like '%term%'` on large tables** without trigram indexes: sequential scans.
- **Extensions unavailable** on your managed service.
- **The application connecting as a superuser or table owner**, bypassing RLS and holding
  far more privilege than it needs.
- **Major version upgrades postponed** until the version is out of support.

---

## 10. How an experienced engineer thinks about this

- **Exhaust PostgreSQL before adding a datastore.** Full-text search, `jsonb`, queues
  and vectors in Postgres are often good enough, with one system to operate.
- **Know the architecture's costs**: per-connection processes, MVCC and vacuum, WAL.
- **Use types and constraints aggressively**; PostgreSQL has more of them than most databases.
- **Least privilege for database roles**, just like for users.
- **Turn on `pg_stat_statements` on day one**; you'll need it.

---

## 11. Check yourself

**Questions**

1. Why does PostgreSQL need connection pooling at scale?
2. What is MVCC, and why does it make VACUUM necessary?
3. When is `jsonb` a good choice, and when should you use columns instead?
4. How does full-text search differ from `like '%term%'`?
5. What do exclusion constraints let you enforce?
6. Why should the application not connect as the schema owner?
7. What are `statement_timeout` and `idle_in_transaction_session_timeout` for?

**Exercises**

1. Implement knowledge-base search with weights and `ts_headline`, and compare results to a
   `like` query.
2. Create the on-call exclusion constraint and try to insert overlapping shifts.
3. Enable row-level security on `tickets` by team, and test it by setting `app.team_id` in
   a transaction.
4. Look at `pg_stat_activity` while running a long query in another session, and cancel it
   with `pg_cancel_backend`.

**Interview-style questions**

- "Why might you choose PostgreSQL over another database?"
- "How would you store and query semi-structured data in a relational database?"
- "How would you implement search for a knowledge base?"

---

## 12. Going deeper

- [PostgreSQL documentation](https://www.postgresql.org/docs/current/) — exceptionally good;
  read the chapters on JSON, full-text search and concurrency control.
- [PostgreSQL 18 release notes](https://www.postgresql.org/docs/release/)
- *The Art of PostgreSQL* by Dimitri Fontaine.
- [Postgres weekly](https://postgresweekly.com/) for keeping up.

**Next:** [Chapter 4 — Indexes](04-indexes.md) explains the single biggest lever on query
performance.
