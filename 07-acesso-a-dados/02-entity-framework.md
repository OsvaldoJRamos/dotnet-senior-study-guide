# Entity Framework

> Geralmente o lazy load faz muitas queries e vira uma bagunça, cai muito a performance. **EVITAR AO MÁXIMO USAR.**

## Lazy Loading

**Definição:** Os dados relacionados **não são carregados automaticamente** quando você consulta a entidade principal. Eles só são buscados no banco **quando acessados pela primeira vez**.

### Exemplo:
```csharp
var blog = context.Blogs.First();
var posts = blog.Posts; // Aqui o EF faz uma nova query para carregar Posts
```

### Vantagens:
- Evita trazer dados desnecessários
- Pode melhorar a performance em cenários simples

### Desvantagens:
- Pode causar o problema do **N+1 queries** (muitas consultas extras)
- Requer que as propriedades de navegação sejam `virtual`
- Em cenários de alta carga, pode gerar gargalo

### Como ativar Lazy Loading:

1. **Propriedades de navegação virtuais:**
```csharp
public class Blog
{
    public int BlogId { get; set; }
    public string Name { get; set; }

    // Precisa ser virtual
    public virtual ICollection<Post> Posts { get; set; }
}
```

2. **Pacote de proxies (no EF Core):**
```bash
dotnet add package Microsoft.EntityFrameworkCore.Proxies
```

3. **Configurar no DbContext:**
```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .UseSqlServer("sua-connection-string")
        .UseLazyLoadingProxies();
}
```

---

## Eager Loading

**Definição:** Os dados relacionados são **carregados junto com a consulta principal** usando `Include`.

### Exemplo:
```csharp
var blog = context.Blogs
    .Include(b => b.Posts)
    .First();
```

Com `ThenInclude` para relações mais profundas:
```csharp
var blogs = context.Blogs
    .Include(b => b.Posts)
        .ThenInclude(p => p.Comments)
    .ToList();
```

Aqui você já carrega tudo de uma vez.

### Vantagens:
- Reduz o número de queries (melhor contra o problema N+1)
- Bom quando você **já sabe** que vai precisar dos dados relacionados

### Desvantagens:
- Pode carregar dados demais, ocupando mais memória e tráfego de rede
- Consultas podem ficar mais pesadas

---

## Quando usar cada um?

| Cenário | Usar |
|---|---|
| Quando nem sempre você precisa dos dados relacionados | Lazy Loading |
| Cenários simples, poucas relações, baixo volume | Lazy Loading |
| Quando você sabe que **sempre vai precisar** das entidades relacionadas | Eager Loading |
| Consultas para APIs que retornam objetos completos (DTOs, ViewModels) | Eager Loading |

## Explicit Loading (alternativa)

Quando você quer controle total sobre quando carregar os dados relacionados:

```csharp
var blog = context.Blogs.First();

// Carrega explicitamente quando necessário
context.Entry(blog)
    .Collection(b => b.Posts)
    .Load();
```

## Boas práticas

- Prefira **Eager Loading** na maioria dos cenários de API
- Use **AsNoTracking()** para queries de leitura (melhor performance):
```csharp
var produtos = context.Produtos.AsNoTracking().ToList();
```
- Evite **Select N+1** — sempre verifique as queries geradas no log

---

[← Anterior: ORM vs Micro ORM vs ADO.NET](01-orm-vs-microorm-vs-adonet.md) | [Voltar ao índice](README.md) | [Próximo: Bancos de Dados →](03-bancos-de-dados.md)
