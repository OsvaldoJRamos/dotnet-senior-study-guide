# Design Patterns

Design patterns from the GoF (Gang of Four) are reusable solutions to common problems in software design.

**Full reference:** [Refactoring Guru - Design Patterns](https://refactoring.guru/pt-br/design-patterns/what-is-pattern)

## Categories

### Creational (how to create objects)

| Pattern | Purpose | Usage example |
|---|---|---|
| **Singleton** | Ensures a single instance of a class | Logger, configuration |
| **Factory Method** | Delegates object creation to subclasses | Creating notifications (email, SMS, push) |
| **Abstract Factory** | Creates families of related objects | Cross-platform UI |
| **Builder** | Builds complex objects step by step | Building queries, complex DTOs |
| **Prototype** | Clones existing objects | Copying configurations |

### Structural (how to compose objects)

| Pattern | Purpose | Usage example |
|---|---|---|
| **Adapter** | Converts the interface of one class into another | Integrating third-party libraries |
| **Decorator** | Adds responsibilities dynamically | Adding logging, caching to services |
| **Facade** | Simplified interface for a complex subsystem | API gateway, facade service |
| **Proxy** | Access controller for an object | Lazy loading, caching, access control |
| **Composite** | Treats individual objects and compositions uniformly | Menus, tree structures |

### Behavioral (how objects communicate)

| Pattern | Purpose | Usage example |
|---|---|---|
| **Strategy** | Defines a family of interchangeable algorithms | Shipping calculation, validations |
| **Observer** | Notifies objects about state changes | Events, pub/sub |
| **Command** | Encapsulates an action as an object | Undo/redo, command queues |
| **Template Method** | Defines algorithm skeleton, subclasses define steps | Data processing |
| **Chain of Responsibility** | Passes request through a chain of handlers | Middleware, validation pipeline |
| **Mediator** | Centralizes communication between objects | MediatR, event bus |

## Most used patterns in day-to-day .NET

### Singleton
```csharp
// In modern .NET, use DI with AddSingleton
builder.Services.AddSingleton<IMyService, MyService>();
```

### Strategy
```csharp
public interface IShippingCalculator
{
    decimal Calculate(Order order);
}

public class StandardShipping : IShippingCalculator { ... }
public class ExpressShipping : IShippingCalculator { ... }

// Usage via DI - the strategy is injected
public class OrderService
{
    private readonly IShippingCalculator _shipping;
    public OrderService(IShippingCalculator shipping) => _shipping = shipping;
}
```

### Repository
```csharp
public interface IRepository<T> where T : class
{
    Task<T?> GetByIdAsync(Guid id);
    Task<IEnumerable<T>> GetAllAsync();
    Task AddAsync(T entity);
    Task UpdateAsync(T entity);
    Task DeleteAsync(Guid id);
}
```

---

[← Previous: SOLID](01-solid.md) | [Back to index](README.md) | [Next: KISS, DRY and YAGNI →](03-kiss-dry-yagni.md)
