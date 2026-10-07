# Database Security and Operations

The database holds the most valuable and sensitive asset of most applications: the data.
It's also the hardest component to recover when something goes wrong. Code can be
redeployed from Git; a dropped table, a corrupted backup or a leaked customer database
can't be undone.

This chapter covers what developers need to know about running a database in production:
access control, protecting data, connections and pooling, backups and recovery, high
availability, monitoring and maintenance. Managed cloud services handle much of the
mechanics, but you still own the decisions.

---

## 1. The problem: the database is the crown jewels

Threats to the database fall into three groups:

| Threat | Examples |
|---|---|
| **Confidentiality** | SQL injection, leaked credentials, over-privileged accounts, unencrypted backups, data copied to dev laptops |
| **Integrity** | Bugs that corrupt data, accidental `delete` without `where`, a bad migration |
| **Availability** | Disk full, connection exhaustion, runaway queries, hardware or zone failure, ransomware |

Security and operations are two sides of the same goal: the data stays correct, private and
available.

---

## 2. Access control: least privilege

Chapter 3 introduced separate roles. The full picture for Beacon:

| Principal | Privileges | Used by |
|---|---|---|
| `beacon_migrator` | Owns the schema; DDL | Deployment pipeline only |
| `beacon_app` | `select/insert/update/delete` on application tables; no DDL | The API |
| `beacon_readonly` | `select` (possibly only on reporting views) | Reporting, BI tools |
| Individual humans | Read-only by default; elevated access via just-in-time, audited process | On-call engineers |
| Superuser / admin | Everything | Break-glass only |

Principles:

- **No shared accounts.** Every human and service has its own identity, so actions are
  attributable and access can be revoked individually.
- **No superuser for applications.** A compromised app shouldn't be able to drop the
  database or read other databases.
- **Passwordless where possible.** Managed identities (Azure) or IAM authentication (AWS)
  let the app connect with short-lived tokens instead of stored passwords (Book IX,
  Chapter 2).
- **Restrict network access.** The database should not be reachable from the internet:
  private networking, firewall rules, or private endpoints (Book IX, Chapter 5).

> **⚠️ What can go wrong:** A database exposed to the internet with a weak password is
> found by scanners within hours. Thousands of PostgreSQL, MongoDB and Redis instances have
> been wiped and held for ransom this way. Never expose a database port publicly, even
> "temporarily."

---

## 3. Protecting the data itself

### Encryption

- **In transit**: require TLS for all connections (`sslmode=verify-full` in Npgsql
  connection strings verifies the server certificate too). Managed services enforce TLS by
  default.
- **At rest**: managed services encrypt storage and backups by default; customer-managed
  keys (in Key Vault) are available when compliance requires them.
- **Column-level encryption** for especially sensitive fields (national IDs, health data):
  encrypt in the application before storing, with keys in a key vault. Trade-off: you can't
  query or index the plaintext.

### Minimizing sensitive data

The safest data is data you don't have:

- Don't store what you don't need (full card numbers: use a payment provider's tokens).
- Store **hashes** for things you only need to compare (API keys, Book III).
- Define **retention**: delete or anonymize data after it's no longer needed (closed tickets
  older than N years), which regulations like GDPR also require.
- Know where personal data lives, including JSON columns, logs and backups.

### Non-production environments

> **⚠️ What can go wrong:** Copying the production database to staging or a developer
> laptop "to reproduce a bug" moves customer data to a less protected environment. Use
> generated data, or a sanitized copy (masked names, emails, free text) produced by a
> reviewed script.

### Auditing

For sensitive tables, record who changed what and when: application-level audit logs (Book
I's `AuditLog`, now persisted), or database-level auditing (`pgaudit` extension) for
compliance needs. Log administrative access to production data.

---

## 4. Connections and pooling

Chapter 3 explained that each PostgreSQL connection is a process. Connection management
is one of the most common operational problems for applications.

### Application-side pooling

Npgsql pools connections per connection string by default: `Open` takes a connection from
the pool; `Dispose` returns it. Key settings:

```text
Host=...;Database=beacon;Username=beacon_app;Maximum Pool Size=50;Timeout=15;Command Timeout=30
```

- **Maximum Pool Size** (default 100) per application instance.
- **Total connections = instances × pool size.** 10 instances × 100 = 1,000 potential
  connections, far above a typical PostgreSQL `max_connections` (100–500 on managed tiers).
  Size pools deliberately.

### Server-side pooling

**PgBouncer** (or the built-in pooler in Azure Database for PostgreSQL Flexible Server)
sits between applications and PostgreSQL, multiplexing many client connections onto fewer
server connections. In **transaction pooling** mode, a server connection is assigned only
for the duration of a transaction. Caveats: session-level features (session-level prepared
statements, `set` without `local`, advisory locks held across transactions, `listen`) don't
work as expected. Npgsql has settings to cooperate with it (e.g. `No Reset On Close=true`
and care with prepared statements).

### How many connections do you need?

Fewer than you'd think. A database server does real work on roughly as many connections as
it has CPU cores (plus some for I/O waits). Hundreds of active queries just contend with
each other. A good pool size per database is often in the low tens to ~100, with queuing
in the pool absorbing bursts.

---

## 5. Backups and recovery

> **🧱 Durable:** A backup you haven't restored is a hope, not a backup. Test restores
> regularly, and measure how long they take.

### Two numbers that drive everything

- **RPO (Recovery Point Objective)**: how much data you can afford to lose. "At most 5
  minutes of tickets."
- **RTO (Recovery Time Objective)**: how long you can be down. "Back within 1 hour."

The business sets these; engineering designs to meet them (and states the cost).

### Backup types

| Type | How | Restores to |
|---|---|---|
| **Logical** | `pg_dump` / `pg_dumpall`: SQL or custom-format export | The time of the dump; portable across versions; slow for large DBs |
| **Physical** | Copy of data files (`pg_basebackup`, storage snapshots) | The time of the backup; fast; same major version |
| **Continuous WAL archiving** | Physical base backup + every WAL segment | **Any point in time** (PITR) |

Managed services provide automated physical backups with **point-in-time restore** over a
retention window (typically 7–35 days). That covers most needs: "restore the database as
it was at 14:02, just before the bad migration."

Additionally:

- **Geo-redundant backups** for region-level disasters.
- **Long-term retention** (monthly/yearly) if regulations require it.
- **Protect backups** from the same credentials as production (ransomware deletes backups
  first): immutable or separately-permissioned storage.
- **Logical dumps** before risky manual operations and for migrating between versions or
  providers.

### Recovering from "oops"

Most data loss isn't hardware failure; it's people and bugs: a `delete` without `where`, a
migration that dropped a column, a bug that overwrote fields. PITR to a *new* server at a
time just before the mistake, then copy the affected rows back, is the standard recovery.
It's much easier if you practiced it.

---

## 6. High availability and scaling

### High availability

**HA** keeps the database available when a server fails:

```text
 Primary (zone 1) ──synchronous replication──► Standby (zone 2)
      │                                          (promoted automatically if the primary fails)
      └─ applications connect via a stable endpoint that follows the primary
```

Managed services offer zone-redundant HA as a setting. Failover takes seconds to a minute,
and **in-flight connections are dropped**, so applications must retry (EF Core's
`EnableRetryOnFailure`, Chapter 7). HA protects against infrastructure failure, **not**
against bad data: a `delete` replicates to the standby immediately. That's what backups are
for.

### Read replicas

**Asynchronous replicas** serve read-only queries: reports, analytics, search. They lag
slightly behind the primary (milliseconds to seconds), so a user who just saved something
may not see it if their next read hits a replica (*read-your-writes* problem). Route
reads that need fresh data to the primary.

### Scaling up and out

In order of complexity:

1. **Optimize queries and indexes** (Chapters 4–6). Usually the biggest win.
2. **Scale up** (bigger instance). Simple, and modern servers are very large.
3. **Read replicas** for read-heavy workloads.
4. **Caching** (Book III, Chapter 7).
5. **Partitioning** large tables (by time, for example) within one database: faster
   maintenance and data retention by dropping old partitions.
6. **Sharding** across multiple databases (Citus, or application-level). A major
   architectural commitment; most applications never need it.

> **🧭 When not to shard:** Until a single, well-tuned, large instance with replicas is
> genuinely insufficient. Sharding complicates queries, transactions, schema changes and
> operations forever.

---

## 7. Monitoring and maintenance

What to watch (Book IX wires these into dashboards and alerts):

| Metric | Why |
|---|---|
| CPU, memory, storage IOPS | Saturation |
| **Storage free space** | A full disk stops writes: alert well before it happens |
| Connections (active, idle, idle in transaction) | Pool sizing, leaks, stuck transactions |
| Replication lag | Stale replicas; HA readiness |
| Slow queries / `pg_stat_statements` top queries | Performance regressions |
| Deadlocks, lock waits | Contention |
| Dead tuples, last autovacuum | Bloat, vacuum falling behind |
| Transaction ID age | Wraparound risk on very busy databases |
| Backup success and age | Silent backup failures |

Routine maintenance:

- **Minor version patches** (security fixes) — automatic or scheduled on managed services.
- **Major version upgrades** at least every few years, before end of support; test the
  application against the new version first.
- **Index maintenance**: drop unused indexes, rebuild bloated ones (`reindex concurrently`).
- **Data retention jobs**: delete or archive old data in batches.
- **Review privileges** periodically.

---

## 8. In practice: Beacon's production database checklist

Beacon will run on Azure Database for PostgreSQL Flexible Server (Book IX). The decisions,
captured as a checklist in the repository (`docs/database.md`):

**Access**
- [x] Roles: `beacon_migrator` (pipeline), `beacon_app` (API, via managed identity),
      `beacon_readonly` (reporting).
- [x] No public network access; private endpoint in the application's virtual network.
- [x] TLS required; `sslmode=verify-full`.
- [x] Human access via Entra ID groups, read-only by default; write access through a
      time-limited, audited elevation.

**Data protection**
- [x] Storage and backup encryption (platform-managed keys; revisit if a customer requires
      customer-managed keys).
- [x] No production data outside production; staging uses generated data from a seeding
      tool.
- [x] Retention: closed tickets anonymized after 3 years (customer-visible fields
      redacted; aggregates kept); job runs nightly in batches of 1,000.

**Connections**
- [x] API pool size 30 per instance; max 6 instances → 180 connections; server
      `max_connections` sized with headroom; built-in PgBouncer enabled for spikes.
- [x] `statement_timeout = 15s`, `idle_in_transaction_session_timeout = 60s` for
      `beacon_app`.

**Recovery**
- [x] RPO 5 minutes, RTO 1 hour (agreed with the product owner).
- [x] Automated backups, 35-day PITR, geo-redundant.
- [x] Quarterly restore drill: restore to a new server, run smoke tests, record time taken.
- [x] Zone-redundant HA enabled; API retries transient failures.

**Monitoring**
- [x] Alerts: storage > 80%, CPU > 80% for 15 min, connections > 80% of max, replication
      lag > 30 s, deadlocks > 0 per hour (warning), backup failures.
- [x] `pg_stat_statements` enabled; weekly review of top queries.

**Change management**
- [x] Migrations run by the pipeline as a separate step, with `lock_timeout = 5s`; failed
      lock acquisition fails the deployment safely rather than queuing traffic.
- [x] Expand-and-contract for breaking schema changes (Chapter 7).

A checklist like this turns many of this book's chapters into decisions someone can review,
and it's exactly what a senior engineer is expected to produce before a system goes live.

---

## 9. What can go wrong

- **Databases reachable from the internet.**
- **Applications connecting as superuser or owner.**
- **Untested backups**, or backups deleted along with production.
- **HA mistaken for backup** (bad data replicates instantly).
- **Connection storms** from too many instances × large pools.
- **Production data copied to insecure environments.**
- **Disk full** from logs, bloat or WAL retained by a stalled replica.
- **Major versions left to expire.**
- **No alerting** until users report the outage.

---

## 10. How an experienced engineer thinks about this

- **Least privilege, private networking, encryption**: the security baseline, not extras.
- **RPO and RTO first**, then design backups and HA to meet them, and test the restore.
- **Connections are a finite, shared resource**; size pools from the server's capacity.
- **Most data loss is human error**; point-in-time restore and practiced recovery are the
  remedy.
- **Write the decisions down.** An operations checklist is part of the system's design.

---

## 11. Check yourself

**Questions**

1. Why should the application not use the schema owner role?
2. What's the difference between RPO and RTO? Who decides them?
3. What can point-in-time recovery do that a nightly `pg_dump` can't?
4. Why isn't high availability a substitute for backups?
5. How do you calculate total potential connections, and why does it matter?
6. What breaks when using PgBouncer in transaction pooling mode?
7. What's the read-your-writes problem with read replicas?

**Exercises**

1. Create the three Beacon roles locally and verify `beacon_app` can't `drop table` or
   `alter table`.
2. Take a `pg_dump` of your local Beacon database, drop a table, and restore it.
3. Write the nightly retention job for anonymizing old closed tickets in batches, using
   `ExecuteUpdate` or SQL with `limit` in a loop.
4. Write your own version of section 8's checklist for a database you work with.

**Interview-style questions**

- "How would you secure a production database?"
- "How do you back up a database, and how do you know the backups work?"
- "How would you scale a database that's running out of capacity?"

---

## 12. Going deeper

- [PostgreSQL documentation: Backup and Restore](https://www.postgresql.org/docs/current/backup.html)
- [PostgreSQL documentation: High Availability](https://www.postgresql.org/docs/current/high-availability.html)
- [Microsoft docs: Azure Database for PostgreSQL – Flexible Server](https://learn.microsoft.com/azure/postgresql/)
- [OWASP Database Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Database_Security_Cheat_Sheet.html)

---

## Book IV wrap-up

Beacon's data now lives in PostgreSQL with a constrained, normalized schema; queries
written in SQL where it's clearer; indexes designed from the workload; concurrency handled
with optimistic tokens, atomic statements and a transactional outbox; slow queries
diagnosed from execution plans; EF Core used as a transparent SQL generator; and a
production checklist for security, backups and operations.

The backend is complete enough to build a real user interface on top of. That starts with
the language underneath it.

**Next:** [Book V — JavaScript & TypeScript](../05-js-ts/README.md).
