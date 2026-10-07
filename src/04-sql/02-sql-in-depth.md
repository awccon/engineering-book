# SQL in Depth

Many application developers know enough SQL to write `SELECT * FROM ... WHERE ...` and let
the ORM handle the rest. That's a real limitation. Reports, dashboards, data fixes,
performance investigations and anything the ORM generates badly all need SQL fluency.
And SQL is one of the most durable skills in software: the SQL you learn today will work
largely unchanged in twenty years.

This chapter covers SQL the way experienced developers use it: thinking in sets,
understanding the logical order of evaluation, joins and their pitfalls, aggregation,
common table expressions and window functions.

---

## 1. The problem: asking questions of data

Beacon's managers want to know:

- Which agents have the most open urgent tickets?
- What's each team's median time to resolution this month?
- Which tickets have had no comment in 48 hours?
- For each ticket, what was the gap between consecutive comments?

Each is one SQL query. Each would be dozens of lines of loops if you loaded the data into
C#, slower, and wrong in subtle ways as data changes during the loop.

---

## 2. The mental model: declarative and set-based

SQL is **declarative**: you describe the result, and the database's **query planner**
decides how to compute it (which indexes, which join algorithms, in which order; Chapter 6).

### Logical order of evaluation

A `SELECT` is written in one order and *logically* evaluated in another:

```text
 Written:                       Logically evaluated:
 SELECT  ...                    1. FROM / JOIN      build the combined rows
 FROM    ...                    2. WHERE            filter rows
 JOIN    ...                    3. GROUP BY         form groups
 WHERE   ...                    4. HAVING           filter groups
 GROUP BY ...                   5. SELECT           compute output columns (incl. window functions)
 HAVING  ...                    6. DISTINCT
 ORDER BY ...                   7. ORDER BY         sort
 LIMIT   ...                    8. LIMIT / OFFSET
```

This explains many "why doesn't this work?" moments:

- You can't use a `SELECT` alias in `WHERE` (it's evaluated before `SELECT`).
- You can't filter on an aggregate in `WHERE`; use `HAVING`.
- Window functions can't appear in `WHERE` (they're computed in step 5); wrap the query.

### NULL: three-valued logic

`NULL` means "unknown." Comparisons with unknown are unknown:

```sql
select null = null;          -- null (not true!)
select 1 where null = null;  -- no rows
select null is null;         -- true
```

So:

- Use `is null` / `is not null`, never `= null`.
- `where assignee_id <> 'maria'` **excludes** unassigned tickets (null <> 'maria' is unknown).
  Write `where assignee_id is distinct from 'maria'` if you want them.
- `not in (subquery)` returns **no rows** if the subquery contains a null. Prefer `not exists`.
- Aggregates ignore nulls: `count(assignee_id)` counts non-null values; `count(*)` counts rows.

---

## 3. Joins

A join combines rows from two tables where a condition holds.

```text
 tickets                 users
 id | assignee_id        id    | display_name
 1  | maria              maria | Maria Lopez
 2  | omar               omar  | Omar Haddad
 3  | null               lee   | Lee Chen
```

| Join | Returns |
|---|---|
| `inner join` | Only matching pairs (tickets 1, 2) |
| `left join` | All left rows; right columns null where no match (tickets 1, 2, 3) |
| `right join` | All right rows (rarely used; swap the tables and use `left`) |
| `full join` | All rows from both sides |
| `cross join` | Every combination (Cartesian product) |

```sql
select t.id, t.title, coalesce(u.display_name, 'Unassigned') as assignee
from tickets t
left join users u on u.id = t.assignee_id
order by t.id;
```

### The left join filter trap

```sql
-- Intended: all tickets, with their comments from maria if any
select t.id, c.body
from tickets t
left join comments c on c.ticket_id = t.id
where c.author_id = 'maria';     -- ✗ turns the left join into an inner join
```

The `where` runs after the join and removes rows where `c.author_id` is null, which are
exactly the tickets without maria's comments. Put conditions on the right table in the
`on` clause:

```sql
left join comments c on c.ticket_id = t.id and c.author_id = 'maria'
```

### Row multiplication

Joining a one-to-many relationship multiplies rows. Joining *two* one-to-many relationships
from the same parent multiplies them by each other:

```sql
-- A ticket with 5 comments and 3 tags produces 15 rows. Counts are now wrong.
select t.id, count(c.id), count(tt.tag_id)
from tickets t
left join comments c on c.ticket_id = t.id
left join ticket_tags tt on tt.ticket_id = t.id
group by t.id;          -- count(c.id) = 15, count(tt.tag_id) = 15  ✗
```

Fix by aggregating each relationship separately (subqueries, lateral joins or CTEs), or
`count(distinct ...)` when appropriate.

### Semi-joins and anti-joins: `exists`

"Tickets that have at least one comment" isn't a join; it's a question about existence:

```sql
select t.* from tickets t
where exists (select 1 from comments c where c.ticket_id = t.id);

-- Anti-join: tickets with no comments
select t.* from tickets t
where not exists (select 1 from comments c where c.ticket_id = t.id);
```

`exists` doesn't multiply rows and stops at the first match. Prefer it over
`join ... distinct` or `in (subquery)` for existence questions.

### Lateral joins

A `lateral` subquery can reference columns from preceding tables, like a `foreach`:

```sql
-- Each ticket with its latest comment
select t.id, t.title, last.body, last.created_at
from tickets t
left join lateral (
    select c.body, c.created_at
    from comments c
    where c.ticket_id = t.id
    order by c.created_at desc
    limit 1
) last on true;
```

---

## 4. Aggregation

```sql
select team_id,
       count(*)                                         as total,
       count(*) filter (where status in (0, 1))         as open,
       count(*) filter (where priority = 3 and status in (0, 1)) as open_urgent,
       avg(resolved_at - created_at) filter (where resolved_at is not null) as avg_resolution
from tickets
group by team_id
having count(*) filter (where status in (0, 1)) > 0
order by open_urgent desc;
```

- Every selected column must be either in `group by` or inside an aggregate.
- `filter (where ...)` (PostgreSQL and the SQL standard) computes conditional aggregates
  without `case` gymnastics.
- `having` filters groups after aggregation.
- Useful aggregates: `count`, `sum`, `avg`, `min`, `max`, `string_agg`, `array_agg`,
  `bool_or`, `percentile_cont(0.5) within group (order by ...)` for medians.

```sql
-- Median resolution time per team this month
select team_id,
       percentile_cont(0.5) within group (order by resolved_at - created_at) as median_resolution
from tickets
where resolved_at >= date_trunc('month', now())
group by team_id;
```

Averages are distorted by outliers; medians and percentiles describe "typical" much
better (the same lesson as latency percentiles in Book III, Chapter 10).

---

## 5. Subqueries and CTEs

A **common table expression** (`with`) names a subquery, making complex queries readable
step by step:

```sql
with open_tickets as (
    select * from tickets where status in (0, 1)
),
last_activity as (
    select t.id, greatest(t.created_at, max(c.created_at)) as last_at
    from open_tickets t
    left join comments c on c.ticket_id = t.id
    group by t.id, t.created_at
)
select t.id, t.title, la.last_at
from open_tickets t
join last_activity la on la.id = t.id
where la.last_at < now() - interval '48 hours'
order by la.last_at;
```

That's "tickets with no activity in 48 hours," readable top to bottom. Since PostgreSQL 12,
CTEs are inlined and optimized like subqueries unless you write `as materialized`.

### Recursive CTEs

For hierarchies (article categories, org charts, threaded comments):

```sql
with recursive category_tree as (
    select id, name, parent_id, 1 as depth, name::text as path
    from categories where parent_id is null
    union all
    select c.id, c.name, c.parent_id, ct.depth + 1, ct.path || ' > ' || c.name
    from categories c
    join category_tree ct on c.parent_id = ct.id
)
select * from category_tree order by path;
```

---

## 6. Window functions

Window functions compute values **across related rows without collapsing them** into
groups. They're one of the most powerful and underused SQL features.

```sql
function(...) over (partition by ... order by ... rows between ...)
```

- `partition by`: like `group by`, but rows aren't collapsed.
- `order by`: the order within each partition.
- frame: which rows relative to the current one are included.

### Ranking

```sql
-- Each agent's three oldest open tickets
select * from (
    select t.*,
           row_number() over (partition by assignee_id order by created_at) as rn
    from tickets t
    where status in (0, 1) and assignee_id is not null
) ranked
where rn <= 3;
```

`row_number` gives 1, 2, 3...; `rank` leaves gaps for ties (1, 1, 3); `dense_rank`
doesn't (1, 1, 2). This "top N per group" pattern is a classic interview question.

### Comparing with neighbors

```sql
-- Time between consecutive comments on each ticket
select ticket_id, created_at,
       created_at - lag(created_at) over (partition by ticket_id order by created_at) as gap
from comments;
```

`lag` / `lead` read the previous or next row; `first_value` / `last_value` read the
edges of the frame.

### Running totals and moving averages

```sql
-- Tickets created per day, with a running total and a 7-day moving average
with daily as (
    select date_trunc('day', created_at)::date as day, count(*) as created
    from tickets
    group by 1
)
select day, created,
       sum(created) over (order by day) as running_total,
       avg(created) over (order by day rows between 6 preceding and current row) as avg_7d
from daily
order by day;
```

### Share of total

```sql
select team_id, count(*) as open,
       round(100.0 * count(*) / sum(count(*)) over (), 1) as pct_of_all_open
from tickets where status in (0, 1)
group by team_id;
```

`sum(count(*)) over ()` sums the per-group counts across all groups: a window over an
aggregate.

---

## 7. Modifying data

### Insert, update, delete with `returning`

```sql
insert into tickets (title, priority, team_id, reporter_id)
values ('VPN drops', 3, 1, 'u-42')
returning id, created_at;

update tickets
set status = 2, resolved_at = now(), version = version + 1
where id = 42 and version = 7          -- optimistic concurrency (Book III, Chapter 4)
returning version;                     -- zero rows returned ⇒ someone else changed it
```

### Upsert

```sql
insert into ticket_tags (ticket_id, tag_id) values (42, 7)
on conflict (ticket_id, tag_id) do nothing;

insert into users (id, display_name, email, role)
values ('u-42', 'Maria Lopez', 'maria@example.com', 'agent')
on conflict (id) do update
set display_name = excluded.display_name, email = excluded.email;
```

`on conflict` is atomic, unlike "select then insert or update" in application code.
PostgreSQL also supports the standard `merge` statement for more complex synchronization.

### Set-based updates

```sql
-- Auto-close tickets resolved more than 7 days ago
update tickets
set status = 3, version = version + 1
where status = 2 and resolved_at < now() - interval '7 days';
```

One statement, atomic, and orders of magnitude faster than loading each ticket into the
application. (Domain events for these closures need thought: Book XIII.)

> **⚠️ What can go wrong:** An `update` or `delete` without a `where` clause affects every
> row. Before running a data fix in production, run it as a `select` with the same `where`
> clause, check the count, then run it inside a transaction (`begin; ... ; select ...;
> commit;` or `rollback;`). Chapter 5 covers transactions.

---

## 8. In practice: Beacon's reporting queries

Here are Beacon's manager reports as SQL. They'll be used by a reporting endpoint
(Chapter 7 shows how to run raw SQL from EF Core).

### Agent workload (Book I, Chapter 7's LINQ report, in SQL)

```sql
select u.display_name as agent,
       count(*)                                                       as active,
       count(*) filter (where now() - t.created_at > case t.priority
                                when 3 then interval '1 hour'
                                when 2 then interval '4 hours'
                                when 1 then interval '1 day'
                                else interval '3 days' end)           as breaching,
       min(t.created_at)                                              as oldest_active
from tickets t
join users u on u.id = t.assignee_id
where t.status in (0, 1)
group by u.display_name
order by breaching desc, active desc;
```

Notice that the SLA rules (`SlaRules.ResponseTarget` in C#) are now duplicated in SQL.
That's a real design tension: Chapter 7 discusses options (an `sla_targets` table both
sides read, or computing in C# after a narrower query).

### First response time per team

How long until a ticket gets its first comment from someone other than the reporter?

```sql
with first_response as (
    select t.id, t.team_id, t.created_at,
           min(c.created_at) as responded_at
    from tickets t
    left join comments c on c.ticket_id = t.id and c.author_id <> t.reporter_id
    where t.created_at >= now() - interval '30 days'
    group by t.id, t.team_id, t.created_at
)
select tm.name as team,
       count(*)                                                                  as tickets,
       count(responded_at)                                                       as responded,
       percentile_cont(0.5) within group (order by responded_at - created_at)    as median_first_response,
       percentile_cont(0.9) within group (order by responded_at - created_at)    as p90_first_response
from first_response fr
join teams tm on tm.id = fr.team_id
group by tm.name
order by median_first_response;
```

### Stale tickets with their latest comment

```sql
select t.id, t.title, u.display_name as assignee, last.created_at as last_comment_at,
       left(last.body, 80) as last_comment
from tickets t
left join users u on u.id = t.assignee_id
left join lateral (
    select c.created_at, c.body from comments c
    where c.ticket_id = t.id
    order by c.created_at desc limit 1
) last on true
where t.status in (0, 1)
  and coalesce(last.created_at, t.created_at) < now() - interval '48 hours'
order by coalesce(last.created_at, t.created_at);
```

Each of these queries returns exactly the rows the report needs, computed where the data
lives.

---

## 9. What can go wrong

- **`= null`** instead of `is null`; `not in` with nulls.
- **Filtering a left join's right table in `where`.**
- **Row multiplication** from multiple one-to-many joins, inflating counts and sums.
- **`select *`** in application queries: fetches unneeded data, breaks when columns are
  added, prevents index-only scans (Chapter 4).
- **Missing `order by`** and relying on "the order it usually comes back in."
- **Unbounded `update`/`delete`.**
- **Row-by-row processing in application loops** instead of set-based statements.
- **Averages where medians or percentiles tell the truth.**

---

## 10. How an experienced engineer thinks about this

- **Think in sets.** Describe the result; let the planner find the path.
- **Know the logical evaluation order.** It explains most SQL errors.
- **Be paranoid about nulls and join cardinality.** Check counts when results look off.
- **Use CTEs for readability** and window functions for "compared to other rows" questions.
- **Verify data fixes before running them**: select first, transaction always.

---

## 11. Check yourself

**Questions**

1. In what logical order is a `SELECT` evaluated? Why can't you use a `SELECT` alias in `WHERE`?
2. Why does `where x <> 'a'` exclude rows where `x` is null?
3. Why can a `where` clause turn a left join into an inner join?
4. When would you use `exists` instead of a join?
5. What's the difference between `group by` and a window function's `partition by`?
6. Explain `row_number`, `rank` and `dense_rank`.
7. Why is `on conflict` safer than select-then-insert?

**Exercises**

1. Write a query for each agent's most recently resolved ticket (window function and lateral
   join versions).
2. Count tickets created per hour of day over the last 30 days, including hours with zero
   tickets (hint: `generate_series`).
3. Find tickets where the same author commented three or more times in a row (hint: `lag`).
4. Write a set-based `update` that assigns all unassigned urgent tickets of a team to its lead.

**Interview-style questions**

- "Find the second highest salary in each department." (Window functions.)
- "What's the difference between `where` and `having`?"
- "Explain inner, left and full outer joins."
- "How do you find rows in one table that have no match in another?"

---

## 12. Going deeper

- [PostgreSQL documentation: Queries](https://www.postgresql.org/docs/current/queries.html)
  and [Window Functions](https://www.postgresql.org/docs/current/tutorial-window.html)
- [Modern SQL](https://modern-sql.com/) by Markus Winand.
- *SQL Performance Explained* by Markus Winand (also free online as "Use The Index, Luke").
- [PostgreSQL Exercises](https://pgexercises.com/) — interactive practice.

**Next:** [Chapter 3 — PostgreSQL Specifics](03-postgresql-specifics.md) covers the
features that make PostgreSQL more than "just SQL."
