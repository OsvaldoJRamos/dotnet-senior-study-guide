# Clean Architecture

## What it is

Architecture proposed by Robert C. Martin (Uncle Bob) that organizes code into **concentric layers**, where **dependencies point inward** -- outer layers depend on inner ones, never the other way around.

```
┌─────────────────────────────────────┐
│          Infrastructure             │  Frameworks, DB, external APIs
│  ┌───────────────────────────────┐  │
│  │        Application            │  │  Use Cases, DTOs, Services
│  │  ┌─────────────────────────┐  │  │
│  │  │        Domain            │  │  │  Entities, Value Objects, Interfaces
│  │  └─────────────────────────┘  │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
```

## Layers

### 1. Domain (center)

The innermost layer. **Does not depend on anything external**. Contains:

- Entities and Aggregates
- Value Objects
- Domain Events
- Repository interfaces (contracts, not implementation)
- Domain Services
- Domain Enums and Exceptions

```csharp
// Domain/Entities/Pedido.cs
public class Pedido
{
    public Guid Id { get; private set; }
    public decimal Total { get; private set; }
    public StatusPedido Status { get; private set; }

    public void Aprovar()
    {
        if (Status != StatusPedido.Pendente)
            throw new DomainException("Só é possível aprovar pedidos pendentes");
        Status = StatusPedido.Aprovado;
    }
}

// Domain/Interfaces/IPedidoRepository.cs
public interface IPedidoRepository
{
    Task<Pedido?> ObterPorIdAsync(Guid id);
    Task SalvarAsync(Pedido pedido);
}
```

### 2. Application (use cases)

Orchestrates the flow. Contains:

- Use Cases / Application Services
- DTOs (input and output)
- External service interfaces
- Input validation
- DTO to entity mapping

```csharp
// Application/UseCases/AprovarPedidoUseCase.cs
public class AprovarPedidoUseCase
{
    private readonly IPedidoRepository _repo;
    private readonly INotificacaoService _notificacao;

    public AprovarPedidoUseCase(IPedidoRepository repo, INotificacaoService notificacao)
    {
        _repo = repo;
        _notificacao = notificacao;
    }

    public async Task ExecutarAsync(Guid pedidoId)
    {
        var pedido = await _repo.ObterPorIdAsync(pedidoId)
            ?? throw new NotFoundException("Pedido não encontrado");

        pedido.Aprovar(); // regra de negocio no dominio

        await _repo.SalvarAsync(pedido);
        await _notificacao.EnviarAsync($"Pedido {pedidoId} aprovado");
    }
}
```

### 3. Infrastructure (boundary)

Concrete implementations. Contains:

- Repositories (EF Core, Dapper)
- External services (email, storage, APIs)
- Database configuration
- ORM mappings

```csharp
// Infrastructure/Repositories/PedidoRepository.cs
public class PedidoRepository : IPedidoRepository
{
    private readonly AppDbContext _context;

    public async Task<Pedido?> ObterPorIdAsync(Guid id)
        => await _context.Pedidos.FindAsync(id);

    public async Task SalvarAsync(Pedido pedido)
    {
        _context.Pedidos.Update(pedido);
        await _context.SaveChangesAsync();
    }
}
```

### 4. Presentation (API/UI)

- Controllers / Minimal APIs
- View Models
- DI configuration
- Middlewares

## Dependency Rule

```
Presentation → Application → Domain ← Infrastructure
                                ↑
                    Infrastructure implements Domain interfaces
```

- **Domain** does not reference any other project
- **Application** references only Domain
- **Infrastructure** references Domain (to implement interfaces)
- **Presentation** references Application and Infrastructure (for DI)

## Typical Solution Structure

```
src/
├── MyApp.Domain/
├── MyApp.Application/
├── MyApp.Infrastructure/
└── MyApp.Api/
tests/
├── MyApp.Domain.Tests/
├── MyApp.Application.Tests/
└── MyApp.Integration.Tests/
```

## Clean Architecture vs others

| Aspect | Clean Architecture | Traditional N-Layer | Vertical Slices |
|---------|-------------------|--------------------|--------------------|
| Dependency | From outside to inside | Top to bottom | Per feature |
| Domain | Center, isolated | Middle, coupled to DB | Inside the feature |
| Testability | High | Medium | High |
| Complexity | Medium-High | Low | Medium |
| When to use | Complex domain | Simple CRUD | Medium domain |

## When NOT to use

- Simple CRUD apps -- overengineering
- Prototypes or MVPs -- too much initial overhead
- Small teams with simple domain -- adds unnecessary layers

---

[← Previous: SAGA Pattern](06-saga-pattern.md) | [Next: CQRS →](08-cqrs.md) | [Back to index](README.md)
