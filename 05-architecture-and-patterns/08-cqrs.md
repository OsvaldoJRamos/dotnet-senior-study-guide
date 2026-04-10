# CQRS - Command Query Responsibility Segregation

## What it is

Separating **read** (Query) and **write** (Command) operations into different models. Instead of a single model that does everything, you have:

- **Command Model**: optimized for writing (normalized, with validations)
- **Query Model**: optimized for reading (denormalized, fast)

```
┌─────────┐    Command    ┌──────────────┐    ┌──────────────┐
│ Cliente  │ ───────────→ │ Command Side │ ──→│  Write DB    │
│          │              │ (validacao,  │    │ (normalizado)│
│          │    Query     │  regras)     │    └──────────────┘
│          │ ───────────→ ├──────────────┤           │ sync
│          │ ←─────────── │  Query Side  │ ←─ ┌──────────────┐
└─────────┘              │ (projecoes)  │    │  Read DB     │
                          └──────────────┘    │(desnormaliz.)│
                                              └──────────────┘
```

## Why use it

1. **Performance**: reads and writes have different needs
2. **Scalability**: scale read and write independently
3. **Simplicity**: each side has a simple model instead of a complex model that tries to do everything
4. **Security**: separate who can read from who can write

## Simple implementation (same database)

You don't need two databases. The most basic level is to separate **handlers**:

```csharp
// Command
public record CriarPedidoCommand(string ClienteId, List<ItemDto> Itens);

public class CriarPedidoHandler
{
    private readonly IPedidoRepository _repo;

    public async Task<Guid> Handle(CriarPedidoCommand cmd)
    {
        var pedido = new Pedido(cmd.ClienteId, cmd.Itens);
        await _repo.SalvarAsync(pedido);
        return pedido.Id;
    }
}

// Query
public record ObterPedidoQuery(Guid PedidoId);

public class ObterPedidoHandler
{
    private readonly IDbConnection _db; // Dapper, leitura direta

    public async Task<PedidoDto?> Handle(ObterPedidoQuery query)
    {
        return await _db.QueryFirstOrDefaultAsync<PedidoDto>(
            "SELECT Id, ClienteNome, Total, Status FROM vw_Pedidos WHERE Id = @Id",
            new { Id = query.PedidoId });
    }
}
```

## With MediatR

```csharp
// Command
public record CriarPedidoCommand(string ClienteId) : IRequest<Guid>;

public class CriarPedidoHandler : IRequestHandler<CriarPedidoCommand, Guid>
{
    public async Task<Guid> Handle(CriarPedidoCommand cmd, CancellationToken ct)
    {
        // ... escrita
    }
}

// Query
public record ObterPedidoQuery(Guid Id) : IRequest<PedidoDto?>;

public class ObterPedidoHandler : IRequestHandler<ObterPedidoQuery, PedidoDto?>
{
    public async Task<PedidoDto?> Handle(ObterPedidoQuery query, CancellationToken ct)
    {
        // ... leitura otimizada
    }
}

// Controller
[HttpPost]
public async Task<IActionResult> Criar(CriarPedidoDto dto)
{
    var id = await _mediator.Send(new CriarPedidoCommand(dto.ClienteId));
    return CreatedAtAction(nameof(Obter), new { id }, null);
}
```

## Event Sourcing (frequently combined with CQRS)

Instead of saving the **current state**, it saves all the **events** that led to the state:

```csharp
// Eventos
public record PedidoCriado(Guid PedidoId, string ClienteId, DateTime Data);
public record ItemAdicionado(Guid PedidoId, string Produto, int Quantidade);
public record PedidoAprovado(Guid PedidoId, DateTime Data);

// O estado e reconstruido "replaying" os eventos:
// PedidoCriado → ItemAdicionado → ItemAdicionado → PedidoAprovado
// Resultado: Pedido com 2 itens, status Aprovado
```

### Event Sourcing Advantages

- **Complete audit trail** -- history of everything that happened
- **Debug** -- can reconstruct state at any point in time
- **Event-driven** -- events can feed other systems

### Disadvantages

- High complexity
- Queries on the event store are slow (requires projections/read models)
- Event versioning is complicated

## CQRS Levels

| Level | Description | Complexity |
|-------|-----------|-------------|
| 1 | Separate read and write handlers | Low |
| 2 | Different models for read and write | Medium |
| 3 | Different databases (write DB + read DB) | High |
| 4 | Event Sourcing + projections | Very high |

> Start at level 1. Only move up if there is a real need.

## When to use

- Complex domain with many write rules
- Read and write with very different volumes
- Need to scale reads independently
- Audit requirements (event sourcing)

## When NOT to use

- Simple CRUDs
- Domain with little business logic
- Small team without experience with the pattern

---

[← Previous: Clean Architecture](07-clean-architecture.md) | [Next: Microservices →](09-microservices.md) | [Back to index](README.md)
