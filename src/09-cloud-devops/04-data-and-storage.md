# Data and Storage

Book IV made Beacon's data correct: a constrained schema, careful transactions, good
indexes. Book VIII ran PostgreSQL in a container. In production, the question changes from
"how do I run a database?" to "which managed data services fit each kind of data, and how
do I configure them safely?"

Azure offers many data services: relational databases, document databases, object storage,
caches, queues, search. This chapter covers the ones most applications need, how to choose
between them, and how Beacon uses them.

---

## 1. The problem: different data, different needs

Beacon stores:

| Data | Shape | Access pattern | Needs |
|---|---|---|---|
| Tickets, comments, users, articles | Relational, constrained | Transactions, queries, reports | ACID, SQL, backups, HA |
| Attachments and exports | Files (KB to hundreds of MB) | Write once, read occasionally | Cheap, durable, secure downloads |
| Cached lookups, SignalR backplane | Key-value, ephemeral | Very fast reads | Low latency, expiry |
| Outbox and background messages | Messages | Produce and consume once | Durability, retries |
| Audit history | Append-only events | Write a lot, read rarely | Cheap, long retention, immutable |
| Search index | Documents + vectors | Text and semantic search | Relevance ranking (Book XI) |

Putting all of this in PostgreSQL is possible (Book IV, Chapter 3 argued for "use what you
have" first). At some point, some of it belongs elsewhere: files almost always do.

---

## 2. The mental model: matching data to stores

| Store type | Azure service | Strengths | Weaknesses |
|---|---|---|---|
| **Relational** | Azure Database for PostgreSQL, Azure SQL Database | Transactions, constraints, joins, SQL | Vertical scaling limits; schema migrations |
| **Document / NoSQL** | Azure Cosmos DB | Global distribution, elastic scale, flexible schema, single-digit-ms latency | No joins; you design partitioning; costs scale with throughput |
| **Object (blob)** | Azure Blob Storage | Very cheap, virtually unlimited, durable files | Not a database: no queries over content |
| **Cache** | Azure Managed Redis | Microsecond-to-millisecond reads, data structures, pub/sub | Memory-bound; data loss acceptable by design |
| **Queue / messaging** | Service Bus, Storage Queues, Event Hubs | Decoupling, buffering, retries | Eventual consistency (Book XIII) |
| **Search** | Azure AI Search | Full-text, facets, vectors, hybrid ranking | A copy of data to keep in sync |
| **Analytics** | Microsoft Fabric, Azure Data Explorer | Large-scale analytical queries | Not for transactional workloads |

> **🧱 Durable:** Choose a data store by **access patterns and consistency requirements**,
> not by fashion. Relational databases remain the right default for business data with
> relationships and invariants; add specialized stores when a specific access pattern
> demands it.

---

## 3. Azure Database for PostgreSQL Flexible Server

The managed PostgreSQL service. Azure handles the server, patching, backups, HA and
monitoring; you manage the database (schema, roles, queries).

### Key configuration decisions

| Decision | Options | Beacon's choice |
|---|---|---|
| **Compute tier** | Burstable (dev), General Purpose, Memory Optimized | General Purpose for prod, Burstable for dev/staging |
| **Storage** | Size and performance tier (IOPS); can auto-grow | Auto-grow on; IOPS sized from measurements (Book VIII, Chapter 7's lesson) |
| **High availability** | None, same-zone, **zone-redundant** (synchronous standby in another zone) | Zone-redundant in prod |
| **Backups** | Automated, PITR 7–35 days, optional geo-redundant | 35 days, geo-redundant (Book IV, Chapter 8's RPO/RTO) |
| **Networking** | Public access with firewall, **private access (VNet integration)**, private endpoints | Private (Chapter 5) |
| **Authentication** | PostgreSQL passwords, **Microsoft Entra**, or both | Entra only for apps and humans (Chapter 2) |
| **Connection pooling** | Built-in PgBouncer | Enabled |
| **Read replicas** | Up to several, same or cross-region | One, for reporting (later) |
| **Maintenance window** | Scheduled patching | Sunday 02:00 local |

### Things that differ from self-hosted PostgreSQL

- **No superuser**: an `azure_pg_admin` role with most, but not all, privileges.
- **Extensions**: allow-listed per server (`azure.extensions` parameter): `pgvector`,
  `pg_trgm`, `pg_stat_statements`, PostGIS and others are available (Book IV, Chapter 3).
- **Server parameters** configured through Azure, not `postgresql.conf`.
- **Failover drops connections**: applications must retry (EF Core's execution strategy,
  Book IV, Chapter 7).

### Azure SQL Database

The managed SQL Server engine, with excellent .NET tooling, serverless auto-pause, Hyperscale
for very large databases, and built-in intelligent tuning. Equally valid for .NET
applications; Beacon uses PostgreSQL because of the choices made in Book IV.

---

## 4. Azure Cosmos DB: when and when not

Cosmos DB is a globally distributed NoSQL database with multiple APIs (NoSQL/document,
MongoDB, PostgreSQL via Citus, Cassandra, Gremlin, Table).

Its core concepts:

- **Containers** of JSON items, each with a **partition key**. Data and throughput are
  distributed across physical partitions by partition key.
- **Request Units (RU/s)**: a normalized cost for operations; you provision throughput
  (or use serverless or autoscale).
- **Tunable consistency**: strong, bounded staleness, session (the default), consistent
  prefix, eventual.
- **Multi-region writes** and guaranteed single-digit-millisecond latency at any scale.

> **🧭 When not to use Cosmos DB:** For relational business data with joins, constraints and
> ad-hoc reporting (most line-of-business apps, including Beacon's tickets), a relational
> database is simpler and cheaper. Cosmos DB shines for massive-scale, globally distributed,
> key-based access patterns: user profiles and sessions at internet scale, IoT telemetry,
> product catalogs, event logs. Its biggest pitfall is a **bad partition key** (a "hot"
> partition, or queries that must fan out across all partitions), which is very hard to
> change later.

---

## 5. Azure Blob Storage

**Blob Storage** stores files (blobs) in **containers** within a **storage account**. It's
cheap, extremely durable (multiple copies; LRS within a data center, ZRS across zones,
GRS/GZRS to a paired region), and scales without limits you'll hit.

### Access tiers and lifecycle

| Tier | Cost to store | Cost to read | Use for |
|---|---|---|---|
| Hot | Highest | Lowest | Frequently accessed |
| Cool / Cold | Lower | Higher | Infrequently accessed (30+ / 90+ days) |
| Archive | Lowest | Highest, hours to rehydrate | Long-term retention, compliance |

**Lifecycle management policies** move blobs between tiers and delete them by age, which is
how Beacon handles "exports expire after 7 days" and "attachments of tickets closed more than
a year ago move to cool storage."

### Secure access patterns

Never make containers public for application data. Options:

- **Entra ID + RBAC**: the app's managed identity has Storage Blob Data Contributor (Chapter 2).
- **User delegation SAS** for direct client access: a short-lived, narrowly scoped URL signed
  with an Entra-derived key (not the account key), letting a browser upload or download one
  blob directly, without passing gigabytes through the API.
- **Disable shared key access** on the storage account where possible, so account keys can't
  be used at all.

### Upload and download flow

```text
 Upload:
 Browser ──POST /api/tickets/T-42/attachments (metadata)──► API: authorize, validate, create pending record
         ◄── { uploadUrl: <user delegation SAS, write-only, 10 min, one blob> } ──
 Browser ──PUT file directly to Blob Storage (SAS URL)──► Blob Storage
         ──POST /api/attachments/{id}/complete──► API: verify blob exists/size/type, mark ready, enqueue malware scan

 Download:
 Browser ──GET /api/attachments/{id}──► API: authorize (Book III, Ch. 6), return 302 to a read-only SAS (5 min)
```

The API stays in control of **authorization**, while large files bypass it entirely.

> **⚠️ What can go wrong:** User-uploaded files are a classic attack vector: malware,
> oversized files, executable content served from your domain (XSS via SVG or HTML uploads),
> path traversal in file names. Validate size and type, store with generated names, serve
> with `Content-Disposition: attachment` and the correct `Content-Type`, scan for malware
> (Defender for Storage offers on-upload scanning), and never trust the client's declared
> file type.

---

## 6. Caches and messaging (briefly)

### Azure Managed Redis

Managed Redis for caching (the L2 behind `HybridCache`, Book III, Chapter 7), session
storage, rate limiting counters, distributed locks, and pub/sub. Choose a tier with
zone redundancy for production; remember that a cache is not a database (data can be
evicted or lost).

> **🔄 Current (as of October 2026):** Azure Managed Redis is the current Redis offering;
> the older Azure Cache for Redis tiers are being retired. Check the migration timeline if you
> run the older service.

### Messaging

- **Azure Service Bus**: enterprise message broker with queues and topics, sessions
  (ordering), dead-lettering, scheduled messages, duplicate detection, transactions. The
  default choice for business messages between services (Book XIII, Chapter 4).
- **Storage Queues**: simple, very cheap queues; fewer features.
- **Event Hubs**: high-throughput event streaming (millions of events per second; Kafka
  compatible), for telemetry and event pipelines.
- **Event Grid**: reactive event routing ("a blob was created" → trigger a function).

Beacon's outbox (Book IV, Chapter 5) can publish to Service Bus when other services need its
events; for in-process workers, the database-backed outbox remains enough.

---

## 7. Data protection and lifecycle

For every data store, decide:

- **Backup and recovery**: PITR for databases; **soft delete**, **versioning** and
  **point-in-time restore** for blob containers; immutable (WORM) policies for audit data.
- **Redundancy**: LRS/ZRS/GZRS for storage; zone-redundant HA for databases.
- **Retention and deletion**: lifecycle policies; legal holds; GDPR erasure across all stores
  (including caches, search indexes, backups, logs).
- **Encryption**: at rest by default with Microsoft-managed keys; customer-managed keys in Key
  Vault where required.
- **Access**: private networking, Entra-only authentication, least-privilege data roles.

---

## 8. In practice: Beacon's data platform on Azure

| Data | Service | Configuration |
|---|---|---|
| Relational data | PostgreSQL Flexible Server `psql-beacon-prod-weu` | General Purpose 4 vCores, zone-redundant HA, 35-day geo-redundant backups, private access, Entra auth, PgBouncer, `pg_stat_statements` + `pg_trgm` + `vector` allowed |
| Attachments, exports | Storage account `stbeaconprodweu` | ZRS, shared key disabled, containers `attachments` (soft delete 14 d, versioning) and `exports` (lifecycle: delete after 7 d), Defender for Storage malware scanning |
| Cache, SignalR | Azure Managed Redis (small, zone-redundant) + Azure SignalR Service | HybridCache L2; Entra auth |
| Audit archive | Storage container `audit` | Immutable (time-based retention 7 years), cool tier |
| Search | PostgreSQL full-text + pgvector (Book XI), Azure AI Search later if needed | — |

Application changes:

- **Attachments feature** using the upload/download flow from section 5, with
  `UserDelegationKey`-based SAS generation via the managed identity:

```csharp
public async Task<Uri> CreateUploadUrlAsync(string blobName, CancellationToken ct)
{
    var service = blobServiceClient;
    var key = await service.GetUserDelegationKeyAsync(DateTimeOffset.UtcNow, DateTimeOffset.UtcNow.AddMinutes(15), ct);
    var sas = new BlobSasBuilder
    {
        BlobContainerName = "attachments",
        BlobName = blobName,                                    // server-generated, e.g. "t-42/7f3c…"
        Resource = "b",
        ExpiresOn = DateTimeOffset.UtcNow.AddMinutes(10),
        Protocol = SasProtocol.Https,
    };
    sas.SetPermissions(BlobSasPermissions.Create | BlobSasPermissions.Write);
    var blob = service.GetBlobContainerClient("attachments").GetBlobClient(blobName);
    return new BlobUriBuilder(blob.Uri) { Sas = sas.ToSasQueryParameters(key.Value, service.AccountName) }.ToUri();
}
```

- **Data Protection keys** stored in a blob and protected with a Key Vault key, shared by all
  BFF and API replicas:

```csharp
builder.Services.AddDataProtection()
    .PersistKeysToAzureBlobStorage(new Uri(cfg["DataProtection:BlobUri"]!), credential)
    .ProtectKeysWithAzureKeyVault(new Uri(cfg["DataProtection:KeyUri"]!), credential)
    .SetApplicationName("beacon");
```

- **Migration from the Book VIII server**: `pg_dump` from the old server, `pg_restore` into
  Flexible Server during a maintenance window (or Azure Database Migration Service for
  near-zero downtime), then switch connection strings. Rehearse in staging first.

---

## 9. What can go wrong

- **Everything in one store** because it's familiar, or **a new store per feature** because
  it's fashionable.
- **Public blob containers** and storage account keys spread through configuration.
- **Uploads proxied through the API**, tying up threads and memory with large files.
- **Unvalidated user uploads.**
- **Cosmos DB with a poor partition key.**
- **No lifecycle policies**: storage costs growing forever.
- **HA without backups**, or backups never restored in practice (Book IV, Chapter 8).
- **Forgetting data in caches, search indexes and logs** when deleting personal data.

---

## 10. How an experienced engineer thinks about this

- **Relational by default; specialized stores for specialized access patterns.**
- **Files belong in object storage**, accessed with short-lived, narrow grants.
- **Every store needs a protection plan**: backup, redundancy, retention, access.
- **Managed services still need configuration**: networking, identity, tiers, limits.
- **Count the copies of each piece of data**, and keep them consistent and deletable.

---

## 11. Check yourself

**Questions**

1. Which Beacon data belongs in PostgreSQL, Blob Storage, Redis and a message broker? Why?
2. What does zone-redundant HA in Flexible Server protect against? What doesn't it protect
   against?
3. When is Cosmos DB a good choice, and what's its biggest design risk?
4. What are Blob Storage access tiers and lifecycle policies for?
5. Why use user delegation SAS for uploads? How does the API keep control of authorization?
6. What risks do user-uploaded files carry, and how do you mitigate them?
7. Why must Data Protection keys be shared across replicas?

**Exercises**

1. Create a Flexible Server with Entra authentication and private access, and connect Beacon.Api
   via a managed identity.
2. Implement Beacon's attachment upload and download with user delegation SAS and test
   authorization for another customer's attachment.
3. Configure a lifecycle policy that deletes exports after 7 days and verify it on test blobs.
4. Migrate a Beacon database from a Docker container to Flexible Server with `pg_dump` and
   `pg_restore`, and time it.

**Interview-style questions**

- "How do you choose between a relational and a NoSQL database?"
- "How would you implement file uploads in a cloud application?"
- "How do you protect data in the cloud against loss and unauthorized access?"

---

## 12. Going deeper

- [Microsoft docs: Azure Database for PostgreSQL – Flexible Server](https://learn.microsoft.com/azure/postgresql/flexible-server/overview)
- [Microsoft docs: Blob Storage security recommendations](https://learn.microsoft.com/azure/storage/blobs/security-recommendations)
- [Microsoft docs: Choose a data store](https://learn.microsoft.com/azure/architecture/guide/technology-choices/data-store-overview)
- [Cosmos DB: Partitioning and horizontal scaling](https://learn.microsoft.com/azure/cosmos-db/partitioning-overview)

**Next:** [Chapter 5 — Networking](05-networking.md) puts all of this on private networks
behind a secure edge.
