# Query Optimization and Execution Plans

"The database is slow" is one of the most common complaints in software, and one of the
most often misdiagnosed. Teams add caches, scale up servers or rewrite in a different
ORM, when the real problem was a single missing index, a stale statistic or a query that
fetched a million rows to display twenty.

This chapter teaches the method: find which queries matter, read the execution plan to see
what the database actually did, understand why, and fix the cause. It ends with a complete
walkthrough of investigating a slow query in Beacon.

---

## 1. The problem: you can't optimize what you can't see

SQL is declarative (Chapter 2): you say *what* you want, and the **planner** decides
*how*. The same query can execute in wildly different ways depending on indexes,
statistics, data distribution and configuration. When it's slow, guessing is useless.
You need to see the plan the database chose and how long each step really took.

---

## 2. The mental model: the planner and the plan

```text
 SQL text ─► Parser ─► Rewriter ─► Planner/Optimizer ─────────────────────► Executor ─► rows
                                   │ considers many plans:
                                   │  - access paths (seq scan, index scan, ...)
                                   │  - join orders
                                   │  - join algorithms
                                   │ estimates each plan's cost from statistics
                                   └► picks the cheapest estimated plan
```

The planner is **cost-based**: it estimates how many rows each step produces
(*cardinality*) and how expensive each step is, using **statistics** about the data (row
counts, distinct values, most common values, histograms), and picks the plan with the
lowest estimated cost.

Most bad plans come from **bad estimates**: the planner thinks a step returns 10 rows and
it returns 500,000, so it picks a strategy that's great for 10 rows and terrible for
500,000.

### Plan nodes

A plan is a tree of nodes. Each node produces rows for its parent:

| Node | What it does |
|---|---|
| Seq Scan | Read the whole table |
| Index Scan / Index Only Scan | Use an index to find rows (Chapter 4) |
| Bitmap Index Scan + Bitmap Heap Scan | Collect row locations, then read pages in order |
| **Nested Loop** | For each row of the outer input, look up matching rows in the inner input. Great when the outer side is small and the inner side has an index |
| **Hash Join** | Build a hash table of one input, probe it with the other. Good for large, unsorted inputs with equality joins |
| **Merge Join** | Merge two inputs sorted on the join key |
| Sort / Incremental Sort | Sort rows (in memory, or spilling to disk) |
| HashAggregate / GroupAggregate | `group by` |
| Limit | Stop after N rows (lets children stop early if they produce rows in order) |
| Gather | Collect rows from parallel workers |

---

## 3. Reading `explain analyze`

`explain` shows the chosen plan with *estimates*. `explain analyze` **runs** the query and
shows *actual* rows and times. Add `buffers` to see how many pages were read:

```sql
explain (analyze, buffers)
select id, title, created_at
from tickets
where team_id = 3 and status in (0, 1)
order by created_at desc
limit 20;
```

```text
Limit  (cost=0.43..28.71 rows=20 width=52) (actual time=0.041..0.093 rows=20 loops=1)
  Buffers: shared hit=24
  ->  Index Scan using tickets_team_open_page on tickets  (cost=0.43..14230.22 rows=10061 width=52)
                                                           (actual time=0.040..0.089 rows=20 loops=1)
        Index Cond: (team_id = 3)
        Buffers: shared hit=24
Planning Time: 0.210 ms
Execution Time: 0.118 ms
```

How to read it:

- **Read from the innermost (most indented) node outward.** Data flows up.
- **`cost=startup..total`**: the planner's estimate in arbitrary units. Useful only for
  comparing plans.
- **`rows=` (estimate) vs `actual ... rows=`**: compare them. A big mismatch (10× or more)
  is the most important clue in any plan.
- **`actual time=first..last`**: milliseconds per loop. Multiply by **`loops`** for the
  node's total time.
- **`Buffers: shared hit`** pages came from PostgreSQL's cache; **`read`** came from disk
  (or the OS cache). Lots of `read` means I/O-bound.
- **The index filtered with `Index Cond`**; a **`Filter:`** line means rows were read and
  then discarded, and `Rows Removed by Filter` tells you how many.

> **⚠️ What can go wrong:** `explain analyze` *executes* the statement. For `insert`,
> `update` or `delete`, wrap it: `begin; explain analyze update ...; rollback;`

---

## 4. The usual suspects

A short list of causes covers most slow queries:

### 1. Missing or unusable index → sequential scan

```text
Seq Scan on tickets  (actual time=0.02..812.4 rows=31 loops=1)
  Filter: (assignee_id = 'maria' AND status = ANY ('{0,1}'))
  Rows Removed by Filter: 1999969
```

Two million rows read, 31 kept. An index on `(assignee_id) where status in (0,1)` turns this
into a few page reads. Or the index exists but the query can't use it (a function on the
column, a type mismatch, a leading wildcard; Chapter 4).

### 2. Bad row estimates

```text
Nested Loop  (rows=12) (actual rows=480211 loops=1)
```

The planner expected 12 rows and chose a nested loop; it got 480,000. Causes:

- **Stale statistics** after bulk loads or big deletes: run `analyze tablename`.
- **Correlated columns**: the planner assumes columns are independent (`city = 'Paris'
  and country = 'France'` is estimated as P(city) × P(country), far too low). Fix with
  extended statistics: `create statistics on city, country from addresses;`.
- **Skewed data**: one customer has 90% of the tickets. Increase the statistics target for
  that column (`alter table ... alter column ... set statistics 1000`).

### 3. Fetching too much

The query returns 50,000 rows to the application, which displays 20. Or `select *`
fetches large text columns that aren't needed. Push filtering, pagination and projection
into the query.

### 4. N+1 queries

Not one slow query, but many fast ones: one query for a list, then one query *per item*
(Book III, Chapter 10's incident). Each query looks fine in isolation; the problem only
appears in traces or query counts per request. Fix with joins, `in (...)` batching or
EF Core's `Include`/projections (Chapter 7).

### 5. Sorts and hashes spilling to disk

```text
Sort Method: external merge  Disk: 184320kB
```

The sort didn't fit in `work_mem` and spilled to disk. Better: avoid the sort with an index
that provides the order. Alternatively, raise `work_mem` for that query or role (carefully:
it's per sort operation, per connection).

### 6. Offset pagination at depth

```sql
select ... order by created_at desc offset 200000 limit 20;
```

Reads and discards 200,000 rows. Use cursor (keyset) pagination (Book III, Chapter 4).

### 7. Lock waits, not slowness

The query is fast when run alone, but waits for locks in production (Chapter 5). Check
`pg_stat_activity.wait_event_type = 'Lock'`.

---

## 5. Finding the queries that matter

Optimize by **total impact**, not by which query looks ugly. `pg_stat_statements`
aggregates statistics for every normalized query:

```sql
select query,
       calls,
       round(total_exec_time::numeric, 0)                as total_ms,
       round(mean_exec_time::numeric, 2)                 as mean_ms,
       rows,
       round(100 * total_exec_time / sum(total_exec_time) over (), 1) as pct_of_total
from pg_stat_statements
order by total_exec_time desc
limit 15;
```

A query averaging 3 ms called 2 million times a day (100 minutes of database time) matters
more than a 4-second report run twice a day (8 seconds). Fix the top of this list first.

Other sources of evidence:

- **`log_min_duration_statement`**: log every query slower than a threshold.
- **`auto_explain`**: automatically log the plans of slow queries.
- **Application traces** (Book IX): show which *requests* issue which queries, and how many.
- **Managed service tools** (Azure's Query Store for PostgreSQL, Performance Insights on AWS).

---

## 6. The optimization method

> **🔍 Investigation: a slow query, step by step.**
> 1. **Reproduce** with realistic data volume and the real parameters. A query that's slow
>    for one large customer may be fast for everyone else.
> 2. **`explain (analyze, buffers)`** and save the output (tools like explain.depesz.com
>    or explain.dalibo.com visualize it).
> 3. **Find where the time goes**: the node with the largest actual time × loops.
> 4. **Compare estimated and actual rows** at each node; find where estimates diverge first.
> 5. **Form a hypothesis**: missing index, bad estimate, too much data, sort spill, N+1.
> 6. **Change one thing** (add an index, `analyze`, rewrite the query) and re-run.
> 7. **Verify the effect on the whole workload**, not just this query: the new index slows
>    writes; the rewrite must return the same results.

Rewrites that frequently help:

- Replace `in (select ...)` / correlated subqueries with `exists` or joins (the planner
  often does this itself, but not always).
- Replace `or` across columns with `union all` of indexable queries.
- Replace functions on columns with ranges.
- Pre-aggregate in a CTE before joining, to avoid multiplying rows (Chapter 2).
- Move filters into the most selective place so fewer rows flow up the plan.

---

## 7. When the query isn't the problem

Sometimes the plan is optimal and the query is still too slow, because the *question* is
expensive: aggregating 100 million rows on every dashboard load. Options beyond tuning:

- **Precompute**: materialized views (`refresh materialized view concurrently` on a
  schedule), summary tables updated incrementally, or counters maintained on write.
- **Cache** the result (Book III, Chapter 7) if staleness is acceptable.
- **Read replicas** to move reporting load off the primary (Chapter 8).
- **A different store** for analytics (a columnar warehouse) at large scale.

> **🧭 When not to optimize:** If a query runs rarely, completes well within its time budget
> and isn't contending with anything, leave it alone. Optimization has costs: indexes slow
> writes, rewrites add complexity, precomputation adds staleness.

---

## 8. In practice: investigating Beacon's slow ticket list

**Symptom.** After a large customer was onboarded, the "My tickets" page for that
customer's users takes 3–4 seconds. Other customers are fine.

**Step 1: Reproduce.** Load a copy of production data (sanitized) or generate realistic
data: 2 million tickets, one customer with 600,000 of them. Capture the exact query from
the logs (EF Core can log SQL; Chapter 7):

```sql
select t.id, t.title, t.status, t.priority, t.created_at,
       (select count(*) from comments c where c.ticket_id = t.id) as comment_count
from tickets t
where t.reporter_id = 'u-big-1' and t.status <> 3
order by t.created_at desc
offset 0 limit 20;
```

**Step 2: Explain.**

```text
Limit  (actual time=3412.5..3412.6 rows=20 loops=1)
  ->  Sort  (actual time=3412.5..3412.5 rows=20 loops=1)
        Sort Key: t.created_at DESC
        Sort Method: top-N heapsort  Memory: 30kB
        ->  Bitmap Heap Scan on tickets t  (rows=2100) (actual rows=541337 loops=1)
              Recheck Cond: (reporter_id = 'u-big-1')
              Filter: (status <> 3)
              Rows Removed by Filter: 58663
              ->  Bitmap Index Scan on tickets_reporter_created (rows=2100) (actual rows=600000)
        SubPlan 1
          ->  Aggregate  (actual time=0.004..0.004 rows=1 loops=541337)
                ->  Index Only Scan using comments_ticket_created on comments c  (loops=541337)
```

**Step 3–4: Where's the time, and where do estimates diverge?**

- Estimated 2,100 rows for this reporter; actual 600,000. The planner thinks this
  customer is typical. (It's skewed data: the statistics' "most common values" list
  doesn't include this new customer yet.)
- Because of the bad estimate, it chose a bitmap scan, then **sorts 541,337 rows** to get
  the top 20, instead of walking the `(reporter_id, created_at desc)` index in order and
  stopping after 20.
- Worse, the **comment-count subquery runs 541,337 times** (`loops=541337`), *before* the
  limit, because the sort needs all rows first.

**Step 5: Hypotheses.** (a) Statistics are stale or too coarse for `reporter_id`. (b) The
`status <> 3` filter prevents the index from providing order cheaply. (c) The subquery
should run only for the 20 returned rows.

**Step 6: Fixes, one at a time.**

1. `analyze tickets;` and raise the statistics target for `reporter_id`. The estimate
   improves to ~590,000 rows. Better, but the plan still sorts.
2. Make the subquery run after the limit, by restructuring:

```sql
with page as (
    select id, title, status, priority, created_at
    from tickets
    where reporter_id = 'u-big-1' and status <> 3
    order by created_at desc
    limit 20
)
select p.*, (select count(*) from comments c where c.ticket_id = p.id) as comment_count
from page p
order by p.created_at desc;
```

3. With `tickets_reporter_created (reporter_id, created_at desc)`, the planner now walks the
   index in order, skips closed tickets with a filter, and stops after 20 matches:

```text
Limit (actual time=0.05..0.31 rows=20)
  ->  Index Scan using tickets_reporter_created on tickets (actual rows=20 loops=1)
        Index Cond: (reporter_id = 'u-big-1')
        Filter: (status <> 3)
        Rows Removed by Filter: 2
SubPlan: loops=20
Execution Time: 0.42 ms
```

From 3.4 seconds to under half a millisecond. And the page is now offset-free: the
endpoint already uses cursor pagination (Book III); this query was from an older
"customer portal" endpoint that used `offset`, which gets migrated too.

**Step 7: Workload check.** No new index was needed; the query rewrite and statistics fixed
it. A query-count and timing test for this endpoint goes into the integration suite with a
skewed dataset, so the regression can't return unnoticed.

---

## 9. What can go wrong

- **Optimizing by guesswork** instead of plans and measurements.
- **Testing with tiny or uniform data**, so plans differ from production.
- **Trusting `explain` estimates** without `analyze` actuals.
- **Ignoring `loops`**: a node that takes 0.01 ms but runs 500,000 times.
- **Fixing one query and slowing the workload** (too many indexes, or a rewrite that's
  worse for other parameter values).
- **Running `explain analyze` on writes** in production without a transaction.
- **Focusing on the slowest query** instead of the one with the most total time.

---

## 10. How an experienced engineer thinks about this

- **Measure the workload, then optimize the biggest consumer.**
- **The plan tells you; don't guess.** Estimates vs actuals is the key comparison.
- **Most fixes are boring**: an index, `analyze`, fetching less, cursor pagination,
  removing N+1.
- **Test with realistic, skewed data**, because production data is never uniform.
- **Prevent regressions** with tests on query counts and performance budgets for critical
  endpoints.

---

## 11. Check yourself

**Questions**

1. What does a cost-based planner do, and what does it depend on?
2. What's the difference between `explain` and `explain analyze`?
3. What does a large gap between estimated and actual rows indicate? Name three causes.
4. When is a nested loop join a good choice, and when is it terrible?
5. What does `Sort Method: external merge Disk:` tell you?
6. How do you find which queries to optimize first?
7. Why can a correlated subquery be dangerous in combination with `order by ... limit`?

**Exercises**

1. Generate 2 million tickets with skewed reporters and reproduce the section 8 walkthrough.
2. Enable `pg_stat_statements`, run Beacon's integration tests against a real database, and
   find the top five queries by total time.
3. Create a query whose estimates are wrong due to correlated columns, then fix it with
   `create statistics`.
4. Paste a plan into explain.dalibo.com and identify the most expensive node.

**Interview-style questions**

- "How do you investigate a slow SQL query?"
- "What's an execution plan? What do you look for in one?"
- "A query is fast in development and slow in production. Why might that be?"

---

## 12. Going deeper

- [PostgreSQL documentation: Using EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [explain.dalibo.com](https://explain.dalibo.com/) and [explain.depesz.com](https://explain.depesz.com/) — plan visualizers.
- *PostgreSQL Query Optimization* by Dombrovskaya, Novikov and Bailliekova.
- [Use The Index, Luke](https://use-the-index-luke.com/).

**Next:** [Chapter 7 — EF Core](07-ef-core.md) connects Beacon to PostgreSQL through the
ORM, using everything in this book so far.
