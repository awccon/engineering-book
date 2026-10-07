# Transactions, Isolation and Locking

Everything works when one user at a time touches the data. Production has hundreds, all
at once. Two agents assign the same ticket simultaneously. A report reads totals while
tickets are being resolved. A transfer debits one account and the server crashes before
crediting the other. A migration locks a table and every request queues behind it.

Transactions, isolation levels and locks are how databases keep data correct under
concurrency and failure. They're also the source of deadlocks, lock contention and
anomalies that appear only under load. This chapter explains them from the ground up and
shows how to use them deliberately.

---

## 1. The problem: concurrency and partial failure

Two separate problems, one mechanism:

1. **Partial failure**: a business operation involves several writes (resolve the ticket,
   insert an audit row, enqueue a notification). If the process crashes halfway, the data
   is left in an inconsistent state.
2. **Concurrent access**: two operations interleave and read or overwrite each other's
   intermediate state: the database version of the race conditions in Book I, Chapter 10.

A **transaction** groups operations into one unit that the database executes with
well-defined guarantees.

---

## 2. The mental model: ACID

| Property | Guarantee | Implemented by |
|---|---|---|
| **Atomicity** | All of the transaction's changes happen, or none do | WAL + rollback |
| **Consistency** | The transaction moves the database from one valid state to another (constraints hold) | Constraints, checked at statement or commit time |
| **Isolation** | Concurrent transactions don't see each other's intermediate states (to a configurable degree) | MVCC + locks |
| **Durability** | Once committed, changes survive crashes | WAL flushed to disk on commit |

```sql
begin;
update tickets set status = 2, resolved_at = now(), version = version + 1 where id = 42;
insert into audit_log (ticket_id, actor_id, action) values (42, 'maria', 'resolved');
insert into outbox (type, payload) values ('TicketResolved', '{"ticketId": 42}');
commit;     -- all three, or (on error / rollback) none
```

Every statement in PostgreSQL runs in a transaction; without `begin`, each statement is its
own transaction (*autocommit*).

---

## 3. MVCC: how PostgreSQL isolates transactions

PostgreSQL uses **Multi-Version Concurrency Control**. Each row version records which
transaction created it and which (if any) deleted it. Each transaction reads from a
**snapshot**: it sees row versions committed before its snapshot was taken, and its own
changes, nothing else.

```text
 time ─►
 T1: begin ─ read ticket 42 (status Open) ───────────────── read ticket 42 again ─ commit
 T2:            begin ─ update 42 → Resolved ─ commit
                        (new row version created; old one kept for T1's snapshot)
```

Consequences:

- **Readers never block writers, and writers never block readers.** A long report doesn't
  stop updates.
- **Writers block writers** on the *same row*: a second `update` of ticket 42 waits until
  the first transaction commits or rolls back.
- Old versions accumulate until no snapshot needs them, then VACUUM reclaims them
  (Chapter 3). A transaction left open for hours keeps every old version alive.

---

## 4. Isolation levels and anomalies

The SQL standard defines **isolation levels** by which **anomalies** they prevent:

| Anomaly | What happens |
|---|---|
| **Dirty read** | Reading another transaction's *uncommitted* changes |
| **Non-repeatable read** | Reading the same row twice and getting different values |
| **Phantom read** | Running the same query twice and getting different *sets* of rows |
| **Lost update** | Two transactions read, modify and write the same value; one change is lost |
| **Write skew** | Two transactions each check a condition, then write different rows, together violating the condition |

| PostgreSQL level | Snapshot taken | Prevents | Notes |
|---|---|---|---|
| **Read Committed** (default) | Per **statement** | Dirty reads | Each statement sees data committed before *it* started |
| **Repeatable Read** | Per **transaction** (first statement) | + non-repeatable reads, phantoms, lost updates (via serialization errors) | Concurrent updates to the same row fail with a serialization error |
| **Serializable** | Per transaction + conflict tracking | All anomalies, including write skew | Transactions may fail with serialization errors and must be **retried** |

(PostgreSQL treats *Read Uncommitted* as Read Committed: dirty reads never happen.)

### Lost update at Read Committed

The classic bug, and the one you'll most likely write:

```text
 T1: select priority from tickets where id = 42;  → 1
 T2: select priority from tickets where id = 42;  → 1
 T1: update tickets set priority = 2 where id = 42;   (escalate: 1 + 1)
 T1: commit
 T2: update tickets set priority = 2 where id = 42;   (escalate: 1 + 1)  — should be 3
 T2: commit
```

Two escalations, one effect. The read-modify-write happened in the application, between
statements. Fixes, from simplest to heaviest:

1. **Do it in one statement**: `update tickets set priority = least(priority + 1, 3) where id = 42`.
   The database serializes concurrent updates to the same row.
2. **Optimistic concurrency**: `update ... where id = 42 and version = 7`; zero rows
   affected means someone else won; reload and retry or report a conflict (Book III's
   ETags). This is what EF Core does with concurrency tokens (Chapter 7).
3. **Pessimistic locking**: `select ... for update` locks the row until commit (section 5).
4. **Repeatable Read or Serializable**: the database detects the conflict and aborts one
   transaction with a serialization error.

### Write skew

A rule: "every team must always have at least one on-call agent." Two agents in a
two-person team both go off call at the same time:

```text
 T1: select count(*) from on_call where team = 1;   → 2   (ok, I can leave)
 T2: select count(*) from on_call where team = 1;   → 2   (ok, I can leave)
 T1: delete from on_call where user_id = 'maria';  commit
 T2: delete from on_call where user_id = 'omar';   commit
 → zero agents on call
```

Neither transaction changed a row the other changed, so row locks and Repeatable Read don't
catch it. Only **Serializable** (or explicit locking of something both transactions
touch, like the team row) prevents it.

> **🧱 Durable:** Most applications run at Read Committed and handle the important
> concurrency cases explicitly (atomic statements, optimistic concurrency, `for update`,
> unique constraints). Use Serializable for transactions with complex invariants across
> rows, and always pair it with **retry logic**, because serialization failures are
> expected, not exceptional.

---

## 5. Locks

### Row-level locks

Writes lock the rows they change until the transaction ends. You can also lock rows
explicitly when reading:

```sql
begin;
select * from tickets where id = 42 for update;    -- others' updates/for update on row 42 now wait
-- ... business logic using the row ...
update tickets set assignee_id = 'maria' where id = 42;
commit;                                             -- lock released
```

| Clause | Blocks |
|---|---|
| `for update` | Other writers and other `for update` / `for share` |
| `for no key update` | Like `for update` but allows concurrent foreign-key checks (what plain `update` takes) |
| `for share` | Writers, but allows other `for share` |
| `nowait` | Fail immediately instead of waiting |
| `skip locked` | Skip rows locked by others (job queues!) |

### `skip locked`: a queue in a table

```sql
-- Each worker claims up to 10 pending jobs no other worker has claimed
with next as (
    select id from jobs
    where status = 'pending' and run_at <= now()
    order by run_at
    limit 10
    for update skip locked
)
update jobs set status = 'running', locked_at = now()
from next where jobs.id = next.id
returning jobs.*;
```

Multiple workers run this concurrently without blocking each other or double-processing.
It's the foundation of many database-backed job systems, and Beacon's outbox processor
(section 8).

### Table-level locks

DDL takes strong locks: `alter table`, `create index` (without `concurrently`), `vacuum
full` and `truncate` can block reads and writes on the whole table. Even quick DDL can
cause an outage if it has to *wait* for a long-running transaction: while it waits, it
blocks everyone queued behind it. Set `lock_timeout` for migrations (Chapter 7).

### Advisory locks

Application-defined locks keyed by a number, not tied to any row:

```sql
select pg_try_advisory_lock(hashtext('sla-monitor'));   -- true if we got it
```

Perfect for "only one instance runs the SLA monitor" (Book III, Chapter 8).

### Deadlocks

```text
 T1: update tickets set ... where id = 1;     (locks 1)
 T2: update tickets set ... where id = 2;     (locks 2)
 T1: update tickets set ... where id = 2;     (waits for T2)
 T2: update tickets set ... where id = 1;     (waits for T1)  → deadlock
```

PostgreSQL detects deadlocks (after `deadlock_timeout`, 1 second by default) and aborts one
transaction with an error. Prevention is the same as in application code: **acquire locks
in a consistent order** (e.g. update rows sorted by ID), keep transactions short, and retry
the aborted transaction.

---

## 6. Transaction design rules

1. **Keep transactions short.** Never wait for user input, HTTP calls or email sending
   inside a transaction. Every second a transaction is open holds locks, blocks vacuum, and
   holds a pooled connection.
2. **Do I/O outside, then write inside.** Fetch remote data first; open the transaction
   only for the writes that must be atomic.
3. **Make retries safe.** Serialization failures (`40001`) and deadlocks (`40P01`) mean
   "try again." Retry the *whole* transaction, which must therefore be idempotent or
   re-read its inputs.
4. **Don't mix a transaction with external side effects.** If you send an email inside a
   transaction and then it rolls back, the email is already sent. This is the dual-write
   problem from Book III, Chapter 8, solved by the outbox.
5. **Set timeouts**: `statement_timeout`, `lock_timeout`, `idle_in_transaction_session_timeout`.

---

## 7. Investigating locking problems

> **🔍 Investigation: "requests are hanging and the database looks idle."**
> Blocked queries show as waiting, not busy. Find who's blocking whom:

```sql
select blocked.pid                      as blocked_pid,
       blocked.query                    as blocked_query,
       now() - blocked.query_start      as waiting_for,
       blocking.pid                     as blocking_pid,
       blocking.state                   as blocking_state,
       blocking.query                   as blocking_query,
       now() - blocking.xact_start      as blocking_xact_age
from pg_stat_activity blocked
join pg_stat_activity blocking
  on blocking.pid = any(pg_blocking_pids(blocked.pid))
order by waiting_for desc;
```

> A common finding is a `blocking_state` of **`idle in transaction`**: a session that
> started a transaction, took locks, and then stopped doing anything, often an application
> bug (an exception path that never committed or rolled back, or a transaction held open
> across an HTTP call). Terminate it with `select pg_terminate_backend(<pid>);`, then fix
> the code and set `idle_in_transaction_session_timeout`.

---

## 8. In practice: Beacon's transactional outbox

Book III left Beacon's notifications in an in-memory queue: lost on crash. Some
notifications must not be lost (emails to customers when their ticket is resolved). The
**transactional outbox** fixes the dual-write problem:

```sql
create table outbox (
    id            bigint generated always as identity primary key,
    type          text not null,
    payload       jsonb not null,
    created_at    timestamptz not null default now(),
    processed_at  timestamptz null,
    attempts      integer not null default 0,
    last_error    text null
);

create index outbox_pending on outbox (id) where processed_at is null;
```

**Writing**: in the *same transaction* as the business change, insert an outbox row.

```sql
begin;
update tickets set status = 2, resolved_at = now(), version = version + 1
where id = 42 and version = 7;                    -- optimistic concurrency; 0 rows → conflict
insert into outbox (type, payload)
values ('TicketResolved', '{"ticketId": 42, "resolvedBy": "maria"}');
commit;
```

Either both happen or neither does. No crash window.

**Processing**: a background worker (Book III, Chapter 8) polls the outbox using
`skip locked`, so several API instances can run it safely:

```sql
-- in a short transaction:
with batch as (
    select id from outbox
    where processed_at is null and attempts < 10
    order by id
    limit 20
    for update skip locked
)
select o.* from outbox o join batch on batch.id = o.id;

-- for each message: publish/send (outside the DB transaction ideally, see below), then:
update outbox set processed_at = now() where id = $1;
-- or on failure:
update outbox set attempts = attempts + 1, last_error = $2 where id = $1;
```

Details that matter:

- **At-least-once.** If the worker sends the email and crashes before marking the row
  processed, the email is sent again on restart. Handlers must be idempotent (Book III,
  Chapter 8), for example by recording the outbox ID with each sent email.
- **Holding the lock while sending** keeps other workers from picking the same row but
  holds a transaction open during I/O. Alternatives: mark rows as "claimed until T+5 min"
  in a short transaction, release, then send. Each approach trades simplicity for
  transaction length.
- **Poison messages**: `attempts < 10` stops retrying forever; a dashboard or alert shows
  messages that exceeded it.
- **Cleanup**: delete processed rows after some days.

Chapter 7 implements this with EF Core, with the outbox row added automatically from
domain events when `SaveChanges` runs, so business code can't forget it.

---

## 9. What can go wrong

- **Lost updates** from read-modify-write in application code.
- **Write skew** under Read Committed or Repeatable Read.
- **Long transactions** holding locks and connections, blocking vacuum.
- **`idle in transaction`** sessions from code paths that forget to commit or roll back.
- **External calls inside transactions** (HTTP, email): slow, and inconsistent on rollback.
- **No retry** for serialization failures and deadlocks.
- **DDL blocking production traffic** while waiting for a lock.
- **Inconsistent lock ordering**, causing deadlocks under load.

---

## 10. How an experienced engineer thinks about this

- **Identify the invariant, then pick the cheapest mechanism that protects it**: a
  constraint, an atomic statement, optimistic concurrency, a row lock, or Serializable.
- **Transactions are for atomicity of writes**, not for wrapping whole requests.
- **Short and boring transactions** cause the fewest problems.
- **Expect conflicts and retries** at higher isolation levels; design for them.
- **Outbox for side effects.** Never rely on "write to the database, then publish."

---

## 11. Check yourself

**Questions**

1. Explain each letter of ACID.
2. How does MVCC let readers and writers avoid blocking each other?
3. What is a lost update? Give three ways to prevent it.
4. What is write skew, and which isolation level prevents it?
5. What do `for update` and `skip locked` do? When would you use `skip locked`?
6. What causes deadlocks, and how does PostgreSQL handle them?
7. Why should HTTP calls never happen inside a database transaction?
8. How does the transactional outbox solve the dual-write problem?

**Exercises**

1. In two `psql` sessions, reproduce a lost update at Read Committed. Then fix it with an
   atomic update, then with a version check.
2. Reproduce write skew with the on-call example at Repeatable Read, then see Serializable
   reject it.
3. Create a deadlock between two sessions and read the error and the server log.
4. Build a tiny job queue table and run three `psql` sessions claiming jobs with
   `skip locked`.

**Interview-style questions**

- "What are transaction isolation levels? Which does your database use by default?"
- "How would you prevent two users from booking the same seat?"
- "What's optimistic vs pessimistic concurrency control?"
- "How do you reliably publish an event when a database row changes?"

---

## 12. Going deeper

- [PostgreSQL documentation: Concurrency Control](https://www.postgresql.org/docs/current/mvcc.html)
- Martin Kleppmann, *Designing Data-Intensive Applications*, chapter 7 ("Transactions") —
  the clearest explanation of isolation anomalies anywhere.
- [Hermitage](https://github.com/ept/hermitage): tests of isolation levels across databases.

**Next:** [Chapter 6 — Query Optimization and Execution Plans](06-query-optimization-and-execution-plans.md)
shows how to find out exactly why a query is slow.
