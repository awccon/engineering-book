# Indexes

If you learn one thing about database performance, learn indexes. A missing index is the
single most common cause of slow applications: a query that takes 2 ms with the right
index takes 2 seconds without it, and the difference only appears once the table grows
past what you tested with. Conversely, indexes added carelessly slow down every write and
waste memory.

This chapter explains what an index physically is, how the database uses it, how to design
indexes for real queries, and when an index hurts more than it helps.

---

## 1. The problem: finding rows without reading everything

Without an index, finding `where assignee_id = 'maria'` in a table of 10 million tickets
means reading every row: a **sequential scan**. That's fine for a 1,000-row table and
disastrous for a large one, especially when the query runs on every page load.

An **index** is a separate data structure, maintained by the database, that lets it find
rows matching a condition without scanning the whole table, the same way a book's index
lets you find a topic without reading every page.

---

## 2. The mental model: the B-tree

PostgreSQL's default index type (and every major database's) is the **B-tree**: a balanced
tree of sorted keys.

```text
                         ┌──────────────────────┐
                         │   [F]        [P]     │      root page
                         └───┬──────────┬────┬──┘
                ┌────────────┘          │    └────────────┐
         ┌──────▼─────┐          ┌──────▼─────┐      ┌────▼───────┐
         │ [B] [D]    │          │ [J] [M]    │      │ [S] [V]    │   internal pages
         └─┬───┬───┬──┘          └────────────┘      └────────────┘
     ┌─────┘   │   └─────┐
 ┌───▼───┐ ┌───▼───┐ ┌───▼───┐
 │A,A,B  │⇄│C,C,D  │⇄│E,E,F  │ ...                                       leaf pages (sorted keys + row pointers),
 └───────┘ └───────┘ └───────┘                                           linked to their neighbors
```

- **Sorted**: keys in the leaves are in order, and leaves are linked, so range scans
  (`between`, `>`, `order by`) just walk the leaves.
- **Balanced and shallow**: every lookup goes root → internal → leaf. A B-tree over 100
  million rows is typically only 3–4 levels deep, so a lookup reads 3–4 pages.
- **Leaves point to table rows** (in PostgreSQL, a tuple ID: page number + offset in the
  table's *heap*).

Finding one row is O(log n), effectively constant. Finding a range is O(log n + matches).

### What an index costs

- **Storage and memory**: each index is another structure competing for cache.
- **Write overhead**: every `insert`, every `delete`, and every `update` touching indexed
  columns must update every relevant index. A table with ten indexes makes writes much
  more expensive.
- **Planning complexity**: more options for the planner to consider (rarely a problem).

> **🧱 Durable:** An index trades **write cost and space** for **read speed** on specific
> query shapes. Every index should exist because of a specific, important query.

---

## 3. How the planner uses indexes

PostgreSQL chooses between several access methods (Chapter 6 shows them in `explain`):

| Scan | How | When chosen |
|---|---|---|
| **Sequential scan** | Read every page of the table | No usable index, or the query returns a large fraction of the table |
| **Index scan** | Walk the index, fetch each matching row from the table | Few matching rows |
| **Index-only scan** | Answer entirely from the index, never touching the table | All needed columns are in the index (and the visibility map is up to date) |
| **Bitmap index scan** | Collect matching row locations from one or more indexes, then read table pages in physical order | Moderate numbers of rows, or combining several indexes |

A crucial and often surprising point: **the planner may correctly ignore your index.** If
a query matches 40% of the table, reading the whole table sequentially is faster than
jumping around via the index. Selectivity matters: indexes help most when the condition
matches a small fraction of rows.

The planner estimates selectivity from **statistics** collected by `analyze` (run by
autovacuum). Stale statistics after a big data load can lead to bad plans; running
`analyze tickets;` fixes that.

---

## 4. Composite indexes and column order

An index can span several columns. Order matters enormously:

```sql
create index tickets_team_status_created on tickets (team_id, status, created_at);
```

The index is sorted by `team_id`, then by `status` within each team, then by `created_at`.
Like a phone book sorted by last name then first name, it efficiently answers queries that
use a **leftmost prefix** of its columns:

| Query condition | Uses the index well? |
|---|---|
| `team_id = 1` | ✓ |
| `team_id = 1 and status = 0` | ✓ |
| `team_id = 1 and status = 0 order by created_at` | ✓✓ (filter and sort both from the index) |
| `team_id = 1 and created_at > ...` | Partly: uses `team_id`, then scans that team's entries |
| `status = 0` | ✗ (not a leftmost prefix; possibly a skip scan in PostgreSQL 18, but don't design for it) |
| `created_at > ...` | ✗ |

Guidelines for column order:

1. **Equality conditions first** (`team_id = ?`, `status = ?`).
2. **Then the range condition or sort column** (`created_at`). Only one range can be used
   efficiently, and it should come last.
3. Among equality columns, the order matters less for single queries, but think about which
   prefixes other queries need.

### Indexes and sorting

An index that matches a query's `order by` lets the database return rows already sorted,
and stop early with `limit`. This is what makes **cursor pagination** (Book III, Chapter 4)
fast:

```sql
-- Beacon's list endpoint
select * from tickets
where team_id = $1 and status in (0, 1)
  and (created_at, id) < ($2, $3)          -- the cursor
order by created_at desc, id desc
limit 21;

create index tickets_team_open_page on tickets (team_id, created_at desc, id desc)
    where status in (0, 1);                -- a partial index; see below
```

The database jumps to the cursor position in the index and reads 21 entries: fast at
page 1 or page 10,000. Offset pagination, by contrast, has to walk past every skipped row.

---

## 5. Specialized indexes

### Partial indexes

Index only the rows that matter:

```sql
create index tickets_open_by_assignee on tickets (assignee_id)
    where status in (0, 1);
```

If 95% of tickets are closed, this index is 20× smaller than a full one, faster to
maintain and more likely to stay in memory. Queries must include a matching condition
(`status in (0, 1)`) for the planner to use it.

Partial unique indexes enforce conditional uniqueness:

```sql
-- Only one published article per slug; drafts may share
create unique index articles_published_slug on articles (slug) where status = 'published';
```

### Expression indexes

Index the result of an expression:

```sql
create index users_email_lower on users (lower(email));
select * from users where lower(email) = lower($1);   -- uses the index
```

The query must use the *same* expression.

### Covering indexes (`include`)

Add non-key columns to the leaf pages, enabling index-only scans:

```sql
create index tickets_assignee_cover on tickets (assignee_id) include (title, priority, status);
-- select title, priority, status from tickets where assignee_id = 'maria'  → index-only scan
```

### Other index types

| Type | For | Example |
|---|---|---|
| **B-tree** | Equality, ranges, sorting (default) | Almost everything |
| **Hash** | Equality only | Rarely better than B-tree |
| **GIN** | "Contains" queries on composite values: `jsonb`, arrays, full-text, trigrams | `metadata @> '{...}'`, `search @@ q`, `name % 'netwrok'` |
| **GiST** | Geometric data, ranges, nearest-neighbor, exclusion constraints | `tstzrange` overlaps, PostGIS |
| **SP-GiST** | Space-partitioned data (IP ranges, some geometric data) | Specialized |
| **BRIN** | Huge tables where values correlate with physical order | `created_at` on an append-only log table: tiny index, good for range scans |
| **HNSW / IVFFlat** (pgvector) | Approximate nearest-neighbor vector search | Embeddings (Book XI) |

---

## 6. When indexes don't help (or can't be used)

- **Functions on indexed columns**: `where date(created_at) = '2026-10-07'` can't use an
  index on `created_at`. Rewrite as a range: `where created_at >= '2026-10-07' and
  created_at < '2026-10-08'`.
- **Leading wildcards**: `like '%vpn'` can't use a B-tree (use `pg_trgm` GIN or full-text
  search).
- **Type mismatches**: comparing a `text` column to an integer parameter, or implicit
  casts, can prevent index use.
- **`or` across different columns** often prevents a single index scan (the planner may use
  a bitmap `or` of two indexes, or rewrite as `union`).
- **Low selectivity**: an index on `status` when 50% of rows are open won't be used for
  "open tickets," and that's correct.
- **Small tables**: a sequential scan of a few pages beats any index.

---

## 7. Designing indexes from queries

Indexes are designed **from the workload**, not from the schema. A practical process:

1. **List the important queries**: frequent ones (every page load) and slow ones.
   `pg_stat_statements` (Chapter 6) ranks them by total time.
2. For each, note the **equality filters, range filters, sort order, and selected columns**.
3. Design the **smallest set of indexes** that serves them, sharing composite indexes
   where leftmost prefixes allow.
4. **Verify** with `explain analyze` on production-like data volumes.
5. **Remove indexes nobody uses**: `pg_stat_user_indexes.idx_scan = 0` over a long period.

### Foreign keys need indexes too

PostgreSQL does *not* automatically index foreign key columns. Without an index on
`comments.ticket_id`:

- loading a ticket's comments scans the whole `comments` table,
- deleting a ticket (with `on delete cascade` or `restrict`) scans `comments` to check for
  references, while holding locks.

Index foreign key columns used in joins or parent deletes, which is nearly all of them.

### Creating indexes in production

`create index` locks the table against writes for the duration of the build, which can take
minutes on a large table. In production, use:

```sql
create index concurrently tickets_open_by_assignee on tickets (assignee_id) where status in (0, 1);
```

It takes longer and can't run inside a transaction, but doesn't block writes. If it fails,
it leaves an `invalid` index that must be dropped and retried. Chapter 7 shows how to do
this in EF Core migrations.

---

## 8. In practice: Beacon's indexes

Beacon's important queries, from Book III's endpoints and Chapter 2's reports:

| Query | Shape |
|---|---|
| Agent's queue | `where team_id = ? and status in (0,1) order by created_at desc, id desc limit ?` (cursor) |
| My assigned tickets | `where assignee_id = ? and status in (0,1)` |
| Customer's tickets | `where reporter_id = ? order by created_at desc` |
| Ticket's comments | `where ticket_id = ? order by created_at` |
| Unread/latest comment per ticket | `where ticket_id = ? order by created_at desc limit 1` |
| Tickets by tag | join `ticket_tags` on `tag_id = ?` |
| Article by slug | `where slug = ?` (unique constraint already indexes it) |
| Article search | `search @@ q` |
| Custom fields | `custom_fields @> ?` |

The resulting index set:

```sql
-- Agent queue: partial (open only), matches filter + sort for cursor pagination
create index concurrently tickets_team_open_page
    on tickets (team_id, created_at desc, id desc) where status in (0, 1);

-- My assigned open tickets
create index concurrently tickets_assignee_open
    on tickets (assignee_id) where status in (0, 1);

-- Customer's tickets, newest first
create index concurrently tickets_reporter_created
    on tickets (reporter_id, created_at desc);

-- Comments of a ticket, in order (serves both "all comments" and "latest comment")
create index concurrently comments_ticket_created
    on comments (ticket_id, created_at);

-- Tickets by tag: the primary key (ticket_id, tag_id) serves ticket→tags; add tag→tickets
create index concurrently ticket_tags_tag on ticket_tags (tag_id, ticket_id);

-- Search and custom fields (Chapter 3)
create index concurrently articles_search_idx on articles using gin (search);
create index concurrently tickets_custom_fields_idx on tickets using gin (custom_fields jsonb_path_ops);
```

Notes:

- `comments_ticket_created` replaces a plain foreign-key index on `comments.ticket_id`
  (its leftmost column is `ticket_id`), so it also covers cascade deletes. A B-tree can be
  scanned backwards, so it serves `order by created_at desc limit 1` too.
- No index on `tickets.status` alone: low selectivity, and the partial indexes cover the
  "open" queries.
- `reporter_id`, `assignee_id` and `team_id` foreign keys are covered by the indexes above
  (leftmost columns), except `team_id` for closed tickets. Deleting a team is rare and
  restricted, so we accept a scan there, deliberately.

Chapter 6 verifies these with `explain analyze` on a million generated tickets.

---

## 9. What can go wrong

- **Missing indexes** on foreign keys and on columns used in frequent `where` and `order by`.
- **Over-indexing**: an index for every column "just in case," slowing writes and wasting
  memory.
- **Wrong column order** in composite indexes.
- **Queries written so indexes can't be used**: functions on columns, leading wildcards,
  type mismatches.
- **Testing with small data**: everything is fast with 1,000 rows; the planner may not even
  use indexes. Test with production-like volumes.
- **`create index` without `concurrently`** on a busy production table, blocking writes.
- **Duplicate and redundant indexes**: `(a)` is redundant if `(a, b)` exists (for most
  purposes).

---

## 10. How an experienced engineer thinks about this

- **Index for queries, not columns.** Start from the workload.
- **Equality first, then range or sort.** The single most useful composite-index rule.
- **Partial indexes for skewed data**, such as "open" tickets in a mostly-closed table.
- **Measure**: `explain analyze` before and after; `pg_stat_user_indexes` for unused indexes.
- **Index changes are deployments.** Plan them like code changes: concurrently, monitored,
  reversible.

---

## 11. Check yourself

**Questions**

1. How does a B-tree index find a row? Why is it only a few levels deep?
2. What costs does an index add?
3. Why might the planner choose a sequential scan even when an index exists?
4. For an index on `(team_id, status, created_at)`, which queries can use it well?
5. What is a partial index, and when is it especially useful?
6. Why doesn't `where date(created_at) = ...` use an index on `created_at`?
7. Why should foreign key columns usually be indexed?
8. Why use `create index concurrently` in production?

**Exercises**

1. Generate a million tickets (`insert ... select ... from generate_series(1, 1000000)`),
   and time the agent-queue query before and after `tickets_team_open_page`.
2. Compare the size of a full index on `assignee_id` and the partial one
   (`pg_relation_size`).
3. Write a query that can't use an index because of a function on the column, then rewrite
   it so it can.
4. Use `pg_stat_user_indexes` to find unused indexes in a database you have access to.

**Interview-style questions**

- "How do database indexes work?"
- "How would you choose the column order for a composite index?"
- "This query is slow. What would you check first?"
- "What are the downsides of adding indexes?"

---

## 12. Going deeper

- [Use The Index, Luke](https://use-the-index-luke.com/) by Markus Winand — the best free
  resource on indexing, for all major databases.
- [PostgreSQL documentation: Indexes](https://www.postgresql.org/docs/current/indexes.html)
- *PostgreSQL Query Optimization* by Henrietta Dombrovskaya et al.

**Next:** [Chapter 5 — Transactions, Isolation and Locking](05-transactions-isolation-and-locking.md)
covers what happens when many users change data at the same time.
