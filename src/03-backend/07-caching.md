# Caching

> *"There are only two hard things in Computer Science: cache invalidation and naming
> things."* — Phil Karlton

Caching is the most effective performance technique available to a backend developer,
and one of the most dangerous. A cache can turn a 200 ms database query into a 0.2 ms
memory lookup. It can also serve stale prices, show one user another user's data, hide a
database outage until the cache expires and then collapse under the load, or make a bug
impossible to reproduce because "it works after a restart."

This chapter covers the kinds of caches, the patterns for using them, the failure modes,
and how to decide whether to cache at all.

---

## 1. The problem: repeated expensive work

Many requests ask for the same data repeatedly:

- The list of ticket categories (changes once a month, read thousands of times a minute).
- A popular knowledge-base article.
- A user's permissions, checked on every request.
- The results of an expensive report.

Recomputing these each time wastes database capacity and adds latency. A **cache** stores
the result of expensive work so subsequent requests can reuse it.

The fundamental trade-off is **freshness vs cost**: cached data may be out of date. The
design question for every cache is *how stale can this data be, and what happens if it is?*

---

## 2. The mental model: a hierarchy of caches

Caches exist at every layer, each closer to the user than the last:

```text
 Browser cache ─► CDN / edge ─► Reverse proxy ─► Output cache ─► App data cache ─► Database (+ its own buffer cache)
 (per user)       (shared,      (shared)         (whole HTTP     (in-memory or      
                   global)                        responses)      distributed)
```

| Layer | Caches | Controlled by | Chapter |
|---|---|---|---|
| Browser | HTTP responses | `Cache-Control`, `ETag` headers | Chapter 1 |
| CDN | Static assets, public responses | Headers + CDN config | Book IX |
| Output cache | Whole endpoint responses | ASP.NET Core middleware | This chapter |
| Data cache | Objects, query results | Your code (`HybridCache`, Redis) | This chapter |
| Database | Pages, query plans | The database | Book IV |

The closer to the user, the bigger the savings (no network trip at all) and the less
control you have over invalidation.

### Key properties of any cache

- **Hit ratio**: the fraction of lookups served from the cache. A cache with a 5% hit ratio
  adds complexity and cost for little benefit.
- **TTL (time to live)**: how long an entry is considered fresh.
- **Eviction policy**: what's removed when the cache is full (LRU: least recently used).
- **Scope**: per-process (in-memory) or shared across instances (distributed).

---

## 3. Caching patterns

### Cache-aside (lazy loading)

The most common pattern: the application checks the cache, and on a miss, loads from the
source and populates the cache.

```text
 read:  cache.get(key) ──hit──► return
                       └─miss─► load from DB ─► cache.set(key, value, ttl) ─► return
 write: update DB ─► invalidate (remove) cache key
```

Simple and resilient (if the cache is down, reads go to the database). The first request
after expiration pays the full cost.

### Read-through / write-through / write-behind

- **Read-through**: the cache itself loads missing data (a library or cache product does
  cache-aside for you).
- **Write-through**: writes go to the cache and the database synchronously.
- **Write-behind**: writes go to the cache, and are flushed to the database later. Fast, but
  risks data loss. Rarely appropriate for business data.

### Invalidation strategies

| Strategy | How | Trade-off |
|---|---|---|
| **TTL only** | Entries expire after a fixed time | Simple; data can be stale up to the TTL |
| **Explicit invalidation** | Remove/update the key when the data changes | Fresh; must find *every* place that changes the data |
| **Tag-based invalidation** | Entries tagged ("ticket:42", "team:network"); evict by tag | Handles many keys depending on one entity |
| **Versioned keys** | Key includes a version (`ticket:42:v7`); bump the version on change | No deletes needed; old entries expire naturally |
| **Event-driven** | Subscribe to change events and invalidate | Works across services; eventual consistency |

In practice, combine **explicit invalidation for correctness** with a **TTL as a safety
net**, so a missed invalidation self-heals eventually.

> **🧱 Durable:** For every cached item, write down: the source of truth, the maximum
> acceptable staleness, and every code path that changes the source. If you can't list
> them, you can't invalidate correctly.

---

## 4. In-memory vs distributed caches

### In-memory (`IMemoryCache`)

Stored in the application process's memory.

- **Fastest possible**: no network, no serialization.
- **Per instance**: with three API instances, each has its own copy. An invalidation on
  one instance doesn't reach the others.
- **Lost on restart** and **consumes the app's memory** (set size limits; Book I, Chapter 11).

### Distributed (`IDistributedCache`: Redis, SQL Server, ...)

A shared cache server.

- **Consistent across instances**: invalidate once, everyone sees it.
- **Survives app restarts and deployments.**
- **Network hop + serialization** on every access (typically sub-millisecond to a few ms).
- **Another piece of infrastructure** to run, secure, monitor and pay for.

**Redis** is the dominant choice. It's also used for rate limiting, distributed locks,
pub/sub, and session storage.

### HybridCache: both, done right

> **🔄 Current (as of October 2026):** `HybridCache` (package
> `Microsoft.Extensions.Caching.Hybrid`, GA since .NET 9) combines a fast in-memory L1
> cache with an optional distributed L2 cache, and adds **stampede protection** and
> **tag-based invalidation**. It's the recommended default for new .NET applications.

```csharp
builder.Services.AddHybridCache(o =>
{
    o.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration = TimeSpan.FromMinutes(5),          // L2 (distributed) lifetime
        LocalCacheExpiration = TimeSpan.FromMinutes(1) // L1 (in-memory) lifetime, shorter
    };
});
// If an IDistributedCache (e.g. Redis via AddStackExchangeRedisCache) is registered, HybridCache uses it as L2.
```

```csharp
var categories = await cache.GetOrCreateAsync(
    "ticket-categories",
    async ct => await db.Categories.AsNoTracking().ToListAsync(ct),
    tags: ["categories"],
    cancellationToken: ct);
```

The shorter L1 expiration limits how long instances can disagree after an invalidation on
another instance.

---

## 5. Failure modes

### Cache stampede (thundering herd)

A popular entry expires. Hundreds of concurrent requests miss at the same moment, and all
of them query the database for the same data simultaneously, potentially overloading it.

Mitigations:

- **Request coalescing**: only one caller loads; others wait for its result. `HybridCache`
  does this automatically per key. (With raw `IMemoryCache`, `GetOrCreateAsync` does *not*
  prevent concurrent factory calls; the `Lazy<Task<T>>` technique from Book I, Chapter 10
  does.)
- **Jittered TTLs**: add randomness to expirations so many keys don't expire together.
- **Refresh ahead**: refresh popular entries in the background before they expire.

### Cache penetration

Requests for data that *doesn't exist* (random IDs, possibly malicious) always miss and
always hit the database. Mitigation: cache negative results ("not found") briefly, and
validate input before lookups.

### Cascading failure on cache loss

If the database is sized for a 95% hit ratio and the cache goes down (or restarts empty),
load on the database jumps 20×. Plan capacity for cache loss, warm caches gradually, and
use circuit breakers.

### Stale or wrong data

- Missed invalidation paths (a background job updates the database directly).
- Race conditions: request A reads old data from the DB, request B updates and invalidates,
  then A writes the old data into the cache. Short TTLs bound the damage; versioned keys
  avoid it.

### Leaking data between users

> **⚠️ What can go wrong:** A cache key that omits the user, tenant or permissions context
> can serve one user's data to another. `"tickets:list:page1"` cached for a customer would
> be served to every customer. Include every input that affects the result in the key
> (`"tickets:list:{userId}:{filters}:{cursor}"`), or don't cache per-user data in shared
> caches at all.

### Caching mutable objects in memory

`IMemoryCache` stores *references*. If a caller modifies a cached object, every later
reader sees the modification. Cache immutable objects (records, frozen collections), or
copies. (`HybridCache` serializes by default, which avoids this for L1 too, unless types
are marked immutable.)

---

## 6. Output caching

ASP.NET Core's **output caching middleware** caches entire HTTP responses server-side:

```csharp
builder.Services.AddOutputCache(o =>
{
    o.AddPolicy("PublicArticles", p => p.Expire(TimeSpan.FromMinutes(10)).Tag("articles"));
});

app.UseOutputCache();

app.MapGet("/api/articles/{slug}", GetArticle).CacheOutput("PublicArticles");

// on publish/update:
await outputCacheStore.EvictByTagAsync("articles", ct);
```

Output caching is ideal for **public, read-heavy** endpoints. By default it doesn't cache
authenticated requests or responses that set cookies, which is the safe default.

Don't confuse it with **response caching** (`UseResponseCaching`), which only honors HTTP
cache headers like a proxy would, and gives the server no control over invalidation.

---

## 7. When not to cache

> **🧭 When not to use it:** Before adding a cache, check whether the underlying operation
> can simply be made fast: a missing index (Book IV, Chapter 4) or an N+1 query (Book IV,
> Chapter 7) is the real cause of most "we need a cache" moments. Caches add staleness,
> invalidation bugs, memory pressure and operational complexity. Cache when the work is
> genuinely expensive, the data is read much more than written, and some staleness is
> acceptable.

Data that's usually **poor** to cache:

- Data that changes on almost every read (low hit ratio).
- Data that must be exactly current (account balances during a transfer, inventory at
  checkout).
- Highly personalized data with low reuse.
- Anything where serving stale data causes real harm (permissions right after revocation).

---

## 8. In practice: caching in Beacon

Beacon has two good candidates:

1. **Published knowledge-base articles**: read constantly by customers, changed rarely,
   public within a tenant. Minutes of staleness is fine.
2. **Agent team membership**: used by authorization on every request, changes rarely, but
   must update promptly when someone moves teams.

And one bad candidate: **ticket lists**. They're per-user, filtered, paginated and change
constantly. Making the query fast (Book IV) is the right approach.

### HybridCache for team membership

```csharp
// src/Beacon.Api/Security/TeamMembershipCache.cs
using Microsoft.Extensions.Caching.Hybrid;

namespace Beacon.Api.Security;

public interface ITeamDirectory
{
    Task<string?> GetTeamAsync(string userId, CancellationToken ct);
}

public sealed class CachedTeamDirectory(ITeamDirectory inner, HybridCache cache) : ITeamDirectory
{
    private static readonly HybridCacheEntryOptions Options = new()
    {
        Expiration = TimeSpan.FromMinutes(10),          // safety net
        LocalCacheExpiration = TimeSpan.FromSeconds(30) // instances converge within 30s
    };

    public async Task<string?> GetTeamAsync(string userId, CancellationToken ct) =>
        await cache.GetOrCreateAsync(
            $"team-of:{userId}",
            (inner, userId),
            static async (state, token) => await state.inner.GetTeamAsync(state.userId, token),
            Options,
            tags: [$"user:{userId}"],
            cancellationToken: ct);

    public ValueTask InvalidateAsync(string userId, CancellationToken ct)
        => cache.RemoveByTagAsync($"user:{userId}", ct);
}
```

Note the `GetOrCreateAsync` overload with a **state** argument and a `static` lambda: no
closure allocation per call (Book I, Chapter 6). This cache sits on every authorized
request, so it's a hot path.

When an admin changes a user's team, the service calls `InvalidateAsync`. If that call is
somehow missed, the 10-minute expiration bounds the staleness.

### Output caching for public articles

```csharp
builder.Services.AddOutputCache(o =>
    o.AddPolicy("Articles", p => p
        .Expire(TimeSpan.FromMinutes(5))
        .SetVaryByRouteValue("slug")
        .Tag("articles")));

app.MapGet("/api/kb/{slug}", GetPublishedArticle)
   .AllowAnonymous()
   .CacheOutput("Articles");
```

When an article is published or edited, the article service evicts the `articles` tag.

### Measuring

Add a counter for hits and misses (Book IX, Chapter 6 exports them as metrics). If the
team-membership hit ratio is below ~90%, the TTLs or the design need revisiting.

---

## 9. What can go wrong

- **Missing invalidation paths**, especially from background jobs, scripts and other services.
- **Cross-user data leaks** from incomplete cache keys.
- **Stampedes** on popular keys.
- **Unbounded in-memory caches** causing memory pressure or OOM kills.
- **Mutating cached objects.**
- **Caching to hide a slow query** that should have been fixed.
- **Different instances showing different data** with in-memory caches and no shared
  invalidation.
- **Debugging confusion**: "it's fixed in the database but users still see the old value."
  Make cache behavior observable (hit/miss metrics, cache headers in dev).

---

## 10. How an experienced engineer thinks about this

- **Fix the slow thing first; cache second.**
- **Define staleness tolerance per data type**, with the business if needed.
- **Every cache entry needs an owner** who knows how it's invalidated.
- **Key = all inputs.** User, tenant, filters, culture, version.
- **Design for cache failure.** The system should be slower, not broken, when the cache is
  empty or down.
- **Measure hit ratios.** A cache that doesn't hit is pure cost.

---

## 11. Check yourself

**Questions**

1. Describe cache-aside. What happens on a miss? On a write?
2. Compare in-memory and distributed caches. When would you use each?
3. What is a cache stampede and how do you prevent it?
4. How can caching leak data between users?
5. What does HybridCache add over `IMemoryCache` and `IDistributedCache`?
6. What's the difference between output caching and response caching?
7. Name three kinds of data you shouldn't cache.

**Exercises**

1. Implement the article output cache with tag eviction on update and verify with timing
   and a debug header that responses come from cache.
2. Simulate a stampede: 200 concurrent requests for an expired key with `IMemoryCache`
   (count factory invocations), then with `HybridCache`.
3. Run Redis in Docker, register it as the L2 cache, run two API instances, and observe
   invalidation behavior across them.
4. Find a cached value in a project you know and write down its source of truth, staleness
   tolerance and every invalidation path.

**Interview-style questions**

- "How would you add caching to a read-heavy API? What could go wrong?"
- "How do you keep a cache consistent with the database?"
- "What happens to your system if Redis goes down?"

---

## 12. Going deeper

- [Microsoft docs: HybridCache](https://learn.microsoft.com/aspnet/core/performance/caching/hybrid)
- [Microsoft docs: Output caching](https://learn.microsoft.com/aspnet/core/performance/caching/output)
- [Redis documentation](https://redis.io/docs/latest/)
- *Designing Data-Intensive Applications* by Martin Kleppmann, for caching in the broader
  context of data systems.

**Next:** [Chapter 8 — Background Work and Real-Time](08-background-work-and-real-time.md)
moves work out of the request path and pushes updates to clients.
