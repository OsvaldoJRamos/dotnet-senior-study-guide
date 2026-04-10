# CQRS - Command Query Responsibility Segregation

## What it is

Separating **read** (Query) and **write** (Command) operations into different models. Instead of a single model that does everything, you have:

- **Command Model**: optimized for writing (normalized, with validations)
- **Query Model**: optimized for reading (denormalized, fast)

```
┌─────────┐    Command    ┌──────────────┐    ┌──────────────┐
│ Client   │ ───────────→ │ Command Side │ ──→│  Write DB    │
│          │              │ (validation, │    │ (normalized) │
│          │    Query     │  rules)      │    └──────────────┘
│          │ ───────────→ ├──────────────┤           │ sync
│          │ ←─────────── │  Query Side  │ ←─ ┌──────────────┐
└─────────┘              │ (projections)│    │  Read DB     │
                          └──────────────┘    │(denormalized)│
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
public record CreateOrderCommand(string CustomerId, List<ItemDto> Items);

public class CreateOrderHandler
{
    private readonly IOrderRepository _repo;

    public async Task<Guid> Handle(CreateOrderCommand cmd)
    {
        var order = new Order(cmd.CustomerId, cmd.Items);
        await _repo.SaveAsync(order);
        return order.Id;
    }
}

// Query
public record GetOrderQuery(Guid OrderId);

public class GetOrderHandler
{
    private readonly IDbConnection _db; // Dapper, direct read

    public async Task<OrderDto?> Handle(GetOrderQuery query)
    {
        return await _db.QueryFirstOrDefaultAsync<OrderDto>(
            "SELECT Id, CustomerName, Total, Status FROM vw_Orders WHERE Id = @Id",
            new { Id = query.OrderId });
    }
}
```

## With MediatR

```csharp
// Command
public record CreateOrderCommand(string CustomerId) : IRequest<Guid>;

public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, Guid>
{
    public async Task<Guid> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        // ... write
    }
}

// Query
public record GetOrderQuery(Guid Id) : IRequest<OrderDto?>;

public class GetOrderHandler : IRequestHandler<GetOrderQuery, OrderDto?>
{
    public async Task<OrderDto?> Handle(GetOrderQuery query, CancellationToken ct)
    {
        // ... optimized read
    }
}

// Controller
[HttpPost]
public async Task<IActionResult> Create(CreateOrderDto dto)
{
    var id = await _mediator.Send(new CreateOrderCommand(dto.CustomerId));
    return CreatedAtAction(nameof(Get), new { id }, null);
}
```

## Event Sourcing (frequently combined with CQRS)

Instead of saving the **current state**, it saves all the **events** that led to the state:

```csharp
// Events
public record OrderCreated(Guid OrderId, string CustomerId, DateTime Date);
public record ItemAdded(Guid OrderId, string Product, int Quantity);
public record OrderApproved(Guid OrderId, DateTime Date);

// The state is reconstructed by "replaying" the events:
// OrderCreated → ItemAdded → ItemAdded → OrderApproved
// Result: Order with 2 items, status Approved
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
