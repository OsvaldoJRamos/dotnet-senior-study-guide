# Caching

## Why use caching

The database is the number 1 bottleneck in web applications. Caching reduces database access and drastically improves response time.

## IMemoryCache (in-process)

Cache in the process memory. Simple and fast, but **not shared** between instances.

```csharp
// Registration
builder.Services.AddMemoryCache();

// Usage
public class ProdutoService
{
    private readonly IMemoryCache _cache;
    private readonly IProdutoRepository _repo;

    public async Task<Produto?> ObterAsync(int id)
    {
        var cacheKey = $"produto:{id}";

        if (_cache.TryGetValue(cacheKey, out Produto? produto))
            return produto;

        produto = await _repo.ObterPorIdAsync(id);

        if (produto != null)
        {
            _cache.Set(cacheKey, produto, new MemoryCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10),
                SlidingExpiration = TimeSpan.FromMinutes(2)
            });
        }

        return produto;
    }
}
```

### GetOrCreate (more concise)

```csharp
var produto = await _cache.GetOrCreateAsync($"produto:{id}", async entry =>
{
    entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
    return await _repo.ObterPorIdAsync(id);
});
```

## IDistributedCache (Redis)

Distributed cache across multiple instances. Survives restarts.

```csharp
// Registration
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
    options.InstanceName = "minha-app:";
});

// Usage
public class ProdutoService
{
    private readonly IDistributedCache _cache;

    public async Task<Produto?> ObterAsync(int id)
    {
        var cacheKey = $"produto:{id}";
        var cached = await _cache.GetStringAsync(cacheKey);

        if (cached != null)
            return JsonSerializer.Deserialize<Produto>(cached);

        var produto = await _repo.ObterPorIdAsync(id);

        if (produto != null)
        {
            await _cache.SetStringAsync(cacheKey,
                JsonSerializer.Serialize(produto),
                new DistributedCacheEntryOptions
                {
                    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(30)
                });
        }

        return produto;
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
[HttpGet("produtos")]
public async Task<IActionResult> Listar()
{
    return Ok(await _service.ListarAsync());
}

// In minimal API
app.MapGet("/produtos", () => service.ListarAsync())
   .CacheOutput(p => p.Expire(TimeSpan.FromMinutes(1)));
```

## Response Caching (HTTP cache headers)

Uses HTTP headers for caching on the **client or CDN**:

```csharp
[ResponseCache(Duration = 300, Location = ResponseCacheLocation.Any)]
[HttpGet("categorias")]
public IActionResult ListarCategorias()
{
    return Ok(categorias);
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
