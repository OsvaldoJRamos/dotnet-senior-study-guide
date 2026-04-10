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
5. **Cache stampede**: when cache expires and N requests hit the database simultaneously. Solution: lock or `GetOrCreate` with factory

---

[← Previous: Background Services](06-background-services.md) | [Next: Minimal APIs →](08-minimal-apis.md) | [Back to index](README.md)
