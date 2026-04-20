# Caching

## Why use caching

The database is the number 1 bottleneck in web applications. Caching reduces database access and drastically improves response time.

## IMemoryCache (in-process)

Cache in the process memory. Simple and fast, but **not shared** between instances.

```csharp
// Registration
builder.Services.AddMemoryCache();

// Usage
public class ProductService
{
    private readonly IMemoryCache _cache;
    private readonly IProductRepository _repo;

    public async Task<Product?> GetAsync(int id)
    {
        var cacheKey = $"product:{id}";

        if (_cache.TryGetValue(cacheKey, out Product? product))
            return product;

        product = await _repo.GetByIdAsync(id);

        if (product != null)
        {
            _cache.Set(cacheKey, product, new MemoryCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),
                SlidingExpiration = TimeSpan.FromMinutes(2)
            });
        }

        return product;
    }
}
```

### GetOrCreate (more concise)

```csharp
var product = await _cache.GetOrCreateAsync($"product:{id}", async entry =>
{
    entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
    return await _repo.GetByIdAsync(id);
});
```

> **`GetOrCreate` / `GetOrCreateAsync` are NOT thread-safe against cache stampede.** If N concurrent requests miss the same key at the same time, the factory delegate runs **N times in parallel** (they each hit the database), and whichever finishes last wins the cache slot. `IMemoryCache` itself is thread-safe for get/set operations, but it does **not** coalesce concurrent factory invocations. Mitigations:
>
> - A `SemaphoreSlim` (or `Lazy<Task<T>>`) **per cache key** to serialize factory execution.
> - The **[LazyCache](https://github.com/alastairtree/LazyCache)** NuGet package — wraps `IMemoryCache` with per-key locking.
> - **HybridCache** (.NET 9+, see below) — solves stampede natively via request coalescing.

## IDistributedCache (Redis)

Distributed cache across multiple instances. Survives restarts.

```csharp
// Registration
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
    options.InstanceName = "my-app:";
});

// Usage
public class ProductService
{
    private readonly IDistributedCache _cache;

    public async Task<Product?> GetAsync(int id)
    {
        var cacheKey = $"product:{id}";
        var cached = await _cache.GetStringAsync(cacheKey);

        if (cached != null)
            return JsonSerializer.Deserialize<Product>(cached);

        var product = await _repo.GetByIdAsync(id);

        if (product != null)
        {
            await _cache.SetStringAsync(cacheKey,
                JsonSerializer.Serialize(product),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
                });
        }

        return product;
    }
}
```

## HybridCache (.NET 9+)

`HybridCache` (from `Microsoft.Extensions.Caching.Hybrid`) is the **recommended caching API going forward**. It unifies in-memory (L1) and distributed (L2) caches behind a single interface and fixes the long-standing gaps in `IMemoryCache` / `IDistributedCache`:

- **L1 + L2 together** — hot reads served from in-process memory; L2 (Redis, SQL, etc.) shares state across instances and survives restarts.
- **Stampede protection** — concurrent requests for the same key share **one** factory execution (request coalescing), no manual locking.
- **Serialization built in** — configurable serializers (JSON by default, MessagePack, Protobuf, etc.).
- **Tag-based invalidation** — `RemoveByTagAsync("products")` invalidates groups of entries.

```csharp
// dotnet add package Microsoft.Extensions.Caching.Hybrid
builder.Services.AddHybridCache(options =>
{
    options.DefaultEntryOptions = new HybridCacheEntryOptions
    {
        Expiration = TimeSpan.FromMinutes(30),      // L2 TTL
        LocalCacheExpiration = TimeSpan.FromMinutes(5) // L1 TTL
    };
});

// Optional: register a distributed L2 (Redis) — HybridCache picks it up automatically
builder.Services.AddStackExchangeRedisCache(o => o.Configuration = "localhost:6379");
```

```csharp
public class ProductService(HybridCache cache, IProductRepository repo)
{
    public Task<Product?> GetAsync(int id, CancellationToken ct) =>
        cache.GetOrCreateAsync(
            $"product:{id}",
            async token => await repo.GetByIdAsync(id, token),
            tags: ["products"],
            cancellationToken: ct);

    public Task InvalidateAllAsync(CancellationToken ct) =>
        cache.RemoveByTagAsync("products", ct).AsTask();
}
```

> Use `HybridCache` for new code on .NET 9+. It replaces most manual `IMemoryCache` + `IDistributedCache` + lock-per-key patterns.

## Output Caching (.NET 7+)

Cache of **entire HTTP responses** on the server:

```csharp
builder.Services.AddOutputCache();
app.UseOutputCache();

// In controller
[OutputCache(Duration = 60)]
[HttpGet("products")]
public async Task<IActionResult> List()
{
    return Ok(await _service.ListAsync());
}

// In minimal API
app.MapGet("/products", () => service.ListAsync())
   .CacheOutput(p => p.Expire(TimeSpan.FromMinutes(1)));
```

## Response Caching (HTTP cache headers)

Uses HTTP headers for caching on the **client or CDN**:

```csharp
[ResponseCache(Duration = 300, Location = ResponseCacheLocation.Any)]
[HttpGet("categories")]
public IActionResult ListCategories()
{
    return Ok(categories);
}

// Generates header: Cache-Control: public, max-age=300
```

## Invalidation strategies

| Strategy | Description | When to use |
|-----------|-----------|-------------|
| **TTL (Time to Live)** | Expires after X minutes | Data that can be stale for a while |
| **Cache-aside** | App checks cache, if missing goes to database | Most common pattern |
| **Write-through** | Write updates cache + database together | Data that changes and is read frequently |
| **Write-behind** | Write updates cache, database is updated later | High write performance |
| **Event-based** | Event invalidates cache when data changes | Stronger consistency |

## IMemoryCache vs IDistributedCache

| Aspect | IMemoryCache | IDistributedCache (Redis) |
|---------|-------------|--------------------------|
| Speed | Faster (in-process) | Slower (network) |
| Sharing | Single instance only | All instances |
| Survives restart | No | Yes |
| Serialization | Not needed | Required (JSON/binary) |
| When to use | App with a single instance | Multiple instances, shared data |

## Tips

1. **Never cache sensitive data** without encryption
2. **Always set a TTL** — cache without expiration is a memory leak
3. **Use cache-aside** as the default pattern
4. **Monitor hit rate** — cache with low hit rate doesn't help
5. **Cache stampede**: when cache expires and N requests hit the database simultaneously. Solution: per-key locking (`SemaphoreSlim`), `LazyCache`, or — preferred on .NET 9+ — `HybridCache`, which coalesces concurrent factory calls automatically

---

[← Previous: Background Services](06-background-services.md) | [Next: Minimal APIs →](08-minimal-apis.md) | [Back to index](README.md)
