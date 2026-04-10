# Caching

## Por que usar cache

Banco de dados e o gargalo numero 1 em aplicacoes web. Cache reduz acessos ao banco e melhora drasticamente o tempo de resposta.

## IMemoryCache (in-process)

Cache na memoria do processo. Simples e rapido, mas **nao compartilha** entre instancias.

```csharp
// Registro
builder.Services.AddMemoryCache();

// Uso
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

### GetOrCreate (mais conciso)

```csharp
var produto = await _cache.GetOrCreateAsync($"produto:{id}", async entry =>
{
    entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10);
    return await _repo.ObterPorIdAsync(id);
});
```

## IDistributedCache (Redis)

Cache distribuido entre multiplas instancias. Sobrevive a restarts.

```csharp
// Registro
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
    options.InstanceName = "minha-app:";
});

// Uso
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

Cache de **respostas HTTP inteiras** no servidor:

```csharp
builder.Services.AddOutputCache();
app.UseOutputCache();

// Em controller
[OutputCache(Duration = 60)]
[HttpGet("produtos")]
public async Task<IActionResult> Listar()
{
    return Ok(await _service.ListarAsync());
}

// Em minimal API
app.MapGet("/produtos", () => service.ListarAsync())
   .CacheOutput(p => p.Expire(TimeSpan.FromMinutes(1)));
```

## Response Caching (HTTP cache headers)

Usa headers HTTP para cache no **cliente ou CDN**:

```csharp
[ResponseCache(Duration = 300, Location = ResponseCacheLocation.Any)]
[HttpGet("categorias")]
public IActionResult ListarCategorias()
{
    return Ok(categorias);
}

// Gera header: Cache-Control: public, max-age=300
```

## Estrategias de invalidacao

| Estrategia | Descricao | Quando usar |
|-----------|-----------|-------------|
| **TTL (Time to Live)** | Expira apos X minutos | Dados que podem ficar stale por um tempo |
| **Cache-aside** | App verifica cache, se nao tem vai ao banco | Padrao mais comum |
| **Write-through** | Escrita atualiza cache + banco juntos | Dados que mudam e sao lidos frequentemente |
| **Write-behind** | Escrita atualiza cache, banco e atualizado depois | Alta performance de escrita |
| **Event-based** | Evento invalida cache quando dado muda | Consistencia mais forte |

## IMemoryCache vs IDistributedCache

| Aspecto | IMemoryCache | IDistributedCache (Redis) |
|---------|-------------|--------------------------|
| Velocidade | Mais rapido (in-process) | Mais lento (rede) |
| Compartilhamento | Uma instancia so | Todas as instancias |
| Sobrevive a restart | Nao | Sim |
| Serializacao | Nao precisa | Precisa (JSON/binary) |
| Quando usar | App com uma instancia | Multiplas instancias, dados compartilhados |

## Dicas

1. **Nunca cache dados sensiveis** sem encriptacao
2. **Defina TTL** sempre — cache sem expiracao e memory leak
3. **Use cache-aside** como padrao
4. **Monitore hit rate** — cache com hit rate baixo nao ajuda
5. **Cache stampede**: quando o cache expira e N requests vao ao banco simultaneamente. Solucao: lock ou `GetOrCreate` com factory

---

[← Anterior: Background Services](06-background-services.md) | [Próximo: Minimal APIs →](08-minimal-apis.md) | [Voltar ao índice](README.md)
