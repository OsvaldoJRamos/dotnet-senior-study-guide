# Caching Strategies

Caches hide latency and reduce load — until they don't. Every cache introduces consistency, invalidation, and failure-mode questions. Know the patterns and their failure modes cold.

## Cache-aside (lazy loading) — the default

The application reads from cache; on miss, it reads from the source and populates the cache.

```csharp
public async Task<Product?> GetProduct(int id)
{
    var key = $"product:{id}";
    var cached = await _cache.GetAsync<Product>(key);
    if (cached is not null) return cached;

    var product = await _repo.GetByIdAsync(id);
    if (product is not null)
        await _cache.SetAsync(key, product, TimeSpan.FromMinutes(10));

    return product;
}
```

| | |
|---|---|
| Pros | Simple, cache failures degrade to direct reads, no write-path coupling |
| Cons | First read is always a miss (cold cache), stale risk on update |
| When | Default for read-heavy workloads with tolerable staleness |

## Read-through

The cache library itself fetches from the source on miss. The app only talks to the cache.

| | |
|---|---|
| Pros | Simpler app code, centralizes the fetch logic |
| Cons | Requires a cache that supports loaders (some client libraries and products do) |
| When | You control the cache layer and want loader logic behind one API |

## Write-through

On write, the app writes to the cache **and** the database synchronously before returning.

| | |
|---|---|
| Pros | Cache never goes stale for the written key |
| Cons | Every write pays the cache hop; write-amplification |
| When | Read-heavy + you want writes to be immediately reflected in the cache |

## Write-back (write-behind)

The app writes to the cache; the cache asynchronously flushes to the database.

| | |
|---|---|
| Pros | Fastest writes, can batch to the DB |
| Cons | Durability risk — a cache crash loses unflushed writes |
| When | Non-critical, recomputable data (view counters, activity counts). **Do not** use for domain events, money, or anything you can't recompute. |

## Cache invalidation

> *"There are only two hard things in Computer Science: cache invalidation and naming things."* — Phil Karlton

Three tools, in increasing order of effort:

1. **TTL (time-to-live)** — the cache entry expires itself. Cheapest, but tolerates staleness up to the TTL.
2. **Explicit invalidate on write** — on update, delete the key. Risk: write succeeds, cache delete fails (or a stale read races in between).
3. **Versioned keys** — bake a version or timestamp into the key (`product:123:v7`). Writing a new version makes old entries instantly irrelevant without a delete step.

### Delete or update?

**Prefer delete over update.** Updating requires you to know the write shape; deleting forces a subsequent miss to refresh from the authoritative source. Update-on-write also races with concurrent cache-asides re-populating stale data.

## TTL strategy

- **Short TTL (seconds-to-minutes)** for mutable data (product prices, inventory, user profiles).
- **Long TTL (hours-to-days)** for immutable or slow-changing data (user settings that change rarely, reference data, static rendered HTML).
- **Add jitter** to TTLs (e.g., `base ± rand(0, 10%)`) to avoid synchronized expirations that cause stampedes (see below).

## Cache stampede (thundering herd)

When a hot key expires, N concurrent readers all miss simultaneously and hammer the source. If the source recomputation is slow, traffic piles up and you cascade-fail.

> Wikipedia: *"A cache stampede is a type of cascading failure that can occur when massively parallel computing systems with caching mechanisms come under a very high load."* ([source](https://en.wikipedia.org/wiki/Cache_stampede))

### Mitigations

| Strategy | How | Trade-off |
|---|---|---|
| **Single-flight / locking** | First miss takes a mutex (e.g., Redis `SET NX PX`), recomputes, releases; others wait or serve stale | Adds a lock hop per recomputation; deadlock risk if the holder crashes — always set a TTL on the lock |
| **Probabilistic early expiration (XFetch)** | Each reader independently decides (with probability rising near expiry) to recompute early, smearing the recompute window | Requires storing computation-time metadata; slightly more code. Cited in Vattani et al. (2015) |
| **Stale-while-revalidate** | Serve stale up to a grace period while a background refresh runs | Staleness risk is explicit; works well with CDNs and HTTP caches (`stale-while-revalidate` Cache-Control directive) |
| **Request coalescing** | The cache tier (or a proxy like Varnish, or fastly's `collapse-forwarding`) merges concurrent misses for the same key | Requires LB/proxy support |

### Single-flight with Redis SET NX

```csharp
// Simplified — production needs timeout handling, retry, and back-off
public async Task<T> GetOrComputeAsync<T>(string key, Func<Task<T>> loader, TimeSpan ttl)
{
    var cached = await _cache.GetAsync<T>(key);
    if (cached is not null) return cached;

    var lockKey = $"{key}:lock";
    // SET NX with expiry — only one caller wins
    var acquired = await _cache.StringSetAsync(lockKey, "1", TimeSpan.FromSeconds(5), When.NotExists);

    if (!acquired)
    {
        // Another caller is recomputing — briefly wait, then re-read cache
        await Task.Delay(50);
        return await _cache.GetAsync<T>(key) ?? await loader();
    }

    try
    {
        var value = await loader();
        await _cache.SetAsync(key, value, ttl);
        return value;
    }
    finally
    {
        await _cache.KeyDeleteAsync(lockKey);
    }
}
```

> Always TTL the lock. A process that crashes holding the lock must not block readers forever.

## Negative caching

Caching misses (*"this key does NOT exist"*) prevents a repeated DB hit on a 404. Use a short TTL (seconds) and a sentinel value or a "does-not-exist" marker. Without this, every lookup for a non-existent user hits the DB — a common DoS vector.

## Redis vs Memcached

Both are in-memory, both are fast, both are commonly deployed as cache sidecars.

| | Redis | Memcached |
|---|---|---|
| Data types | Strings, hashes, lists, sets, sorted sets, streams, HLL, geospatial, pub/sub | Strings only |
| Persistence | Optional (RDB snapshots, AOF log) | None |
| Replication | Primary + replicas, Redis Sentinel, Redis Cluster | None built-in |
| Clustering | Redis Cluster (hash-slot sharding) | Client-side sharding only |
| Threading | Mostly single-threaded for commands; I/O threads in newer versions | Multi-threaded from day one |
| Memory efficiency | Good, but type overhead | Slightly better for simple KV |
| Eviction policies | Multiple (`allkeys-lru`, `volatile-ttl`, `allkeys-lfu`, etc.) | LRU-based |
| Use cases | Cache, session store, rate limiter, queue, leaderboard, pub/sub | Pure key-value cache |

**Default to Redis.** The feature gap is huge, managed services are everywhere (ElastiCache, Azure Cache for Redis, Upstash), and you almost always want at least one non-string data type eventually. Pick Memcached only when you specifically want a pure-KV cache with Memcached's multi-threading model.

## HybridCache (.NET 9+)

`Microsoft.Extensions.Caching.Hybrid` (shipped with ASP.NET Core 9.0) combines a primary in-process cache with an optional secondary out-of-process cache (e.g., Redis via `IDistributedCache`) behind one API, with stampede protection built in — *"the HybridCache service ensures that only one concurrent caller for a given key calls the factory method"* ([docs](https://learn.microsoft.com/en-us/aspnet/core/performance/caching/hybrid)). It replaces hand-rolled two-tier caches and is a drop-in addition over existing `IDistributedCache` backends; the library also targets .NET Framework 4.7.2 / .NET Standard 2.0 per the compatibility section of the docs.

## CDN caching

CDNs are caches at the network edge, controlled primarily by HTTP headers on your origin responses.

### Key response headers

| Header | Effect |
|---|---|
| `Cache-Control: public, max-age=3600` | Cache at any cache (CDN, browser) for 1 hour |
| `Cache-Control: private, max-age=60` | Only the end-user browser — CDNs will NOT cache |
| `Cache-Control: no-store` | Never cache anywhere |
| `Cache-Control: stale-while-revalidate=30` | Serve stale up to 30 s while async revalidating |
| `ETag` / `If-None-Match` | Conditional requests; 304 Not Modified avoids re-transfer |
| `Vary: Accept-Encoding, Accept-Language` | Multiple cache variants keyed on these headers |

### Cache keys and the Vary trap

By default a CDN keys by scheme + host + path + query. A `Vary: User-Agent` explodes cache cardinality (thousands of UA strings = thousands of variants). Keep `Vary` tight — `Accept-Encoding` is essentially always needed; everything else is suspect.

### Invalidation strategies on a CDN

- **TTL expiry** (simplest).
- **Explicit purge** via provider API (expensive, rate-limited — Cloudflare, Fastly, CloudFront all have purge APIs).
- **Versioned URLs** (`/assets/app.a3f9c1.js`) — cheapest and safest: update the HTML to reference the new URL; the CDN serves the old URL to whoever still has the stale HTML, and there's no purge needed.

## What NOT to cache

- Highly personalized per-user data that changes per request.
- Security-sensitive responses without `private` + short TTL (avoid serving cached auth'd pages to the wrong user).
- Anything where staleness could cause a monetary or safety issue (balance, inventory close to zero, medical data) without explicit invalidation.

---

[← Previous: Load Balancing](04-load-balancing.md) | [Next: Rate Limiting →](06-rate-limiting.md) | [Back to index](README.md)
