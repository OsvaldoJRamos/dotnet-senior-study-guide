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
// Domain/Entities/Order.cs
public class Order
{
    public Guid Id { get; private set; }
    public decimal Total { get; private set; }
    public OrderStatus Status { get; private set; }

    public void Approve()
    {
        if (Status != OrderStatus.Pending)
            throw new DomainException("Only pending orders can be approved");
        Status = OrderStatus.Approved;
    }
}

// Domain/Interfaces/IOrderRepository.cs
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(Guid id);
    Task SaveAsync(Order order);
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
// Application/UseCases/ApproveOrderUseCase.cs
public class ApproveOrderUseCase
{
    private readonly IOrderRepository _repo;
    private readonly INotificationService _notification;

    public ApproveOrderUseCase(IOrderRepository repo, INotificationService notification)
    {
        _repo = repo;
        _notification = notification;
    }

    public async Task ExecuteAsync(Guid orderId)
    {
        var order = await _repo.GetByIdAsync(orderId)
            ?? throw new NotFoundException("Order not found");

        order.Approve(); // business rule in the domain

        await _repo.SaveAsync(order);
        await _notification.SendAsync($"Order {orderId} approved");
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
// Infrastructure/Repositories/OrderRepository.cs
public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;

    public async Task<Order?> GetByIdAsync(Guid id)
        => await _context.Orders.FindAsync(id);

    public async Task SaveAsync(Order order)
    {
        _context.Orders.Update(order);
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
