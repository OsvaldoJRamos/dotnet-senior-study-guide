# Architecture and Patterns

> Read the questions, think about your answer, then click to reveal.

---

### 1. What is the Single Responsibility Principle (SRP)?

<details>
<summary>Reveal answer</summary>

A class should have **one, and only one, reason to change** — meaning it should have only one responsibility or concern.

**Violation example**: a `UserService` that validates input, saves to the database, sends emails, and generates reports.

**Fix**: split into `UserValidator`, `UserRepository`, `EmailService`, `ReportGenerator`. Each class has one reason to change.

SRP is not about "doing one thing" — it's about having **one actor** (stakeholder) whose requirements drive changes to the class.

Deep dive: [SOLID](../05-architecture-and-patterns/01-solid.md)

</details>

---

### 2. What is the Open/Closed Principle (OCP)?

<details>
<summary>Reveal answer</summary>

Software entities should be **open for extension but closed for modification**. You should be able to add new behavior without changing existing, tested code.

```csharp
// Bad — modifying existing code for every new discount type
if (type == "Premium") discount = 0.20m;
else if (type == "VIP") discount = 0.30m;
// Adding a new type requires modifying this method

// Good — extending via abstraction
public interface IDiscountStrategy { decimal Calculate(Order order); }
public class PremiumDiscount : IDiscountStrategy { ... }
public class VipDiscount : IDiscountStrategy { ... }
// Adding a new discount = adding a new class, no existing code changes
```

Key enablers: interfaces, abstract classes, Strategy pattern, plugin architectures.

Deep dive: [SOLID](../05-architecture-and-patterns/01-solid.md)

</details>

---

### 3. What is the Liskov Substitution Principle (LSP)?

<details>
<summary>Reveal answer</summary>

Subtypes must be **substitutable for their base types** without breaking the program's correctness.

**Classic violation**: `Square` inheriting from `Rectangle`. Setting width on a `Square` also changes height, violating the contract that `Rectangle` consumers expect.

**Practical rules:**
- Don't throw `NotImplementedException` in overridden methods
- Don't strengthen preconditions (require more than the base)
- Don't weaken postconditions (deliver less than the base)
- Subclass behavior should not surprise consumers of the base type

Deep dive: [SOLID](../05-architecture-and-patterns/01-solid.md)

</details>

---

### 4. What is the Interface Segregation Principle (ISP)?

<details>
<summary>Reveal answer</summary>

Clients should not be forced to depend on interfaces they do not use. Prefer **small, focused interfaces** over large, monolithic ones.

```csharp
// Bad — fat interface
public interface IWorker
{
    void Work();
    void Eat();
    void Sleep();
}
// A Robot implements IWorker but doesn't eat or sleep

// Good — segregated
public interface IWorkable { void Work(); }
public interface IFeedable { void Eat(); }
public class Robot : IWorkable { ... }
public class Human : IWorkable, IFeedable { ... }
```

ISP helps keep dependencies lean and makes mocking in tests easier.

Deep dive: [SOLID](../05-architecture-and-patterns/01-solid.md)

</details>

---

### 5. What is the Dependency Inversion Principle (DIP)?

<details>
<summary>Reveal answer</summary>

High-level modules should not depend on low-level modules. Both should depend on **abstractions**. Abstractions should not depend on details — details should depend on abstractions.

```csharp
// Bad — high-level depends on low-level
class OrderService
{
    private readonly SqlOrderRepository _repo = new(); // tight coupling
}

// Good — both depend on abstraction
class OrderService
{
    private readonly IOrderRepository _repo;
    public OrderService(IOrderRepository repo) => _repo = repo;
}
```

DIP is the foundation of **Dependency Injection**. It enables testability (mock the interface), flexibility (swap implementations), and follows Clean Architecture's dependency rule.

Deep dive: [SOLID](../05-architecture-and-patterns/01-solid.md)

</details>

---

### 6. Explain Clean Architecture and its dependency rule.

<details>
<summary>Reveal answer</summary>

Code organized in **concentric layers** where dependencies always point **inward**:

```
Presentation → Application → Domain ← Infrastructure
```

| Layer | Contains | Depends on |
|-------|----------|-----------|
| **Domain** (center) | Entities, value objects, repository interfaces | Nothing |
| **Application** | Use cases, DTOs, orchestration | Domain |
| **Infrastructure** | EF Core, external APIs, implementations | Domain (implements interfaces) |
| **Presentation** | Controllers, API endpoints | Application + Infrastructure (for DI wiring) |

The **dependency rule**: source code dependencies can only point inward. The Domain layer never references Infrastructure. Infrastructure implements Domain interfaces (Dependency Inversion).

Deep dive: [Clean Architecture](../05-architecture-and-patterns/07-clean-architecture.md)

</details>

---

### 7. What is CQRS and at what levels can you apply it?

<details>
<summary>Reveal answer</summary>

**CQRS** (Command Query Responsibility Segregation) separates **read** (Query) and **write** (Command) operations into different models.

| Level | Description | Complexity |
|-------|-------------|------------|
| 1 | Separate handlers for reads and writes (same DB, same model) | Low |
| 2 | Different models for read and write (same DB) | Medium |
| 3 | Different databases (write DB + read-optimized DB) | High |
| 4 | Event Sourcing + projections | Very high |

**When to use**: when read and write patterns differ significantly (e.g., complex writes but simple, high-volume reads). **Don't use** for simple CRUDs.

Start at level 1. Only move up if there is a real, measured need.

Deep dive: [CQRS](../05-architecture-and-patterns/08-cqrs.md)

</details>

---

### 8. When would you choose microservices vs a modular monolith?

<details>
<summary>Reveal answer</summary>

| Scenario | Recommendation |
|----------|----------------|
| New project, small team | Monolith or Modular Monolith |
| Simple domain, mostly CRUD | Monolith |
| Complex domain, medium team | Modular Monolith |
| Multiple teams, independent scaling needs | Microservices |
| Need independent deployments | Microservices |

The **Modular Monolith** gives you most microservice benefits (bounded contexts, independent modules) without the operational cost (networking, distributed transactions, observability complexity).

> "If you can't build a well-made monolith, you won't be able to build microservices." — Simon Brown

**Path**: start with a Modular Monolith, extract to microservices when a specific module needs independent scaling or deployment.

Deep dive: [Microservices](../05-architecture-and-patterns/09-microservices.md)

</details>

---

### 9. What is the SAGA pattern? What is the difference between orchestration and choreography?

<details>
<summary>Reveal answer</summary>

SAGA manages **distributed transactions** across multiple services by breaking them into a sequence of local transactions with compensating actions for rollback.

| Style | How it works | Pros | Cons |
|-------|-------------|------|------|
| **Orchestration** | A central orchestrator directs each step | Easy to understand, centralized logic | Single point of failure, coupling to orchestrator |
| **Choreography** | Each service listens to events and acts independently | Loosely coupled, no central coordinator | Hard to track, implicit flow, debugging is harder |

**Example** (order flow): Create Order → Reserve Stock → Charge Payment → Ship.
If payment fails: Compensate Stock → Cancel Order.

Use **orchestration** when the flow is complex and visibility matters. Use **choreography** for simpler, more decoupled flows.

Deep dive: [SAGA Pattern](../05-architecture-and-patterns/06-saga-pattern.md)

</details>

---

### 10. Explain the Strategy pattern. When would you use it?

<details>
<summary>Reveal answer</summary>

The Strategy pattern defines a **family of algorithms**, encapsulates each one, and makes them interchangeable at runtime.

```csharp
public interface IShippingCalculator
{
    decimal Calculate(Order order);
}

public class StandardShipping : IShippingCalculator
{
    public decimal Calculate(Order order) => 9.99m;
}

public class ExpressShipping : IShippingCalculator
{
    public decimal Calculate(Order order) => 29.99m;
}

// Usage — strategy is injected
public class OrderService(IShippingCalculator shipping)
{
    public decimal GetTotal(Order order) =>
        order.Subtotal + shipping.Calculate(order);
}
```

**When to use**: when you have multiple ways to do the same thing and want to switch between them without modifying the consuming code. Eliminates `if/else` or `switch` chains.

Deep dive: [Design Patterns](../05-architecture-and-patterns/02-design-patterns.md)

</details>

---

### 11. Explain the Factory pattern. What problem does it solve?

<details>
<summary>Reveal answer</summary>

The Factory pattern **encapsulates object creation** — the client gets the object it needs without knowing the concrete class or creation logic.

```csharp
public interface INotificationSender { void Send(string message); }

public class NotificationFactory
{
    public INotificationSender Create(string channel) => channel switch
    {
        "email" => new EmailSender(),
        "sms"   => new SmsSender(),
        "push"  => new PushSender(),
        _       => throw new ArgumentException($"Unknown channel: {channel}")
    };
}
```

**Problems it solves:**
- Decouples creation from usage (consumer depends on abstraction)
- Centralizes complex creation logic
- Makes it easy to add new types without modifying consumers
- Works well with DI — register the factory, not every concrete type

Deep dive: [Design Patterns](../05-architecture-and-patterns/02-design-patterns.md)

</details>

---

### 12. What is the Observer pattern? How does it relate to C# events?

<details>
<summary>Reveal answer</summary>

The Observer pattern defines a **one-to-many relationship** where when one object (subject) changes state, all its dependents (observers) are notified automatically.

C# **events** are a built-in implementation of Observer:

```csharp
// Subject (publisher)
public class OrderService
{
    public event EventHandler<Order>? OrderPlaced;

    public void PlaceOrder(Order order)
    {
        // ... save order
        OrderPlaced?.Invoke(this, order); // notify all observers
    }
}

// Observer (subscriber)
orderService.OrderPlaced += (sender, order) => SendEmail(order);
orderService.OrderPlaced += (sender, order) => UpdateInventory(order);
```

Modern alternatives: MediatR notifications, `IObservable<T>` (Reactive Extensions), or message brokers for cross-service observation.

Deep dive: [Design Patterns](../05-architecture-and-patterns/02-design-patterns.md)

</details>

---

### 13. What are some common anti-patterns and how do you avoid them?

<details>
<summary>Reveal answer</summary>

| Anti-pattern | What it is | Fix |
|-------------|-----------|-----|
| **God Class** | One class does everything (thousands of lines) | Split by responsibility (SRP) |
| **Golden Hammer** | Using the same technology/pattern for everything | Choose tools based on the problem |
| **Premature Optimization** | Optimizing before measuring | Profile first, optimize bottlenecks |
| **Spaghetti Code** | No clear structure, tangled dependencies | Apply architecture layers, SOLID |
| **Shotgun Surgery** | One change requires modifying many files | Group related logic together (cohesion) |
| **Anemic Domain Model** | Entities with only getters/setters, logic in services | Move behavior into domain objects (DDD) |
| **Service Locator** | Resolving dependencies at runtime instead of injecting | Use constructor injection |

Deep dive: [KISS, DRY, YAGNI](../05-architecture-and-patterns/03-kiss-dry-yagni.md)

</details>

---

### 14. What are the key DDD concepts: aggregate, value object, and bounded context?

<details>
<summary>Reveal answer</summary>

| Concept | Definition | Example |
|---------|-----------|---------|
| **Aggregate** | A cluster of entities treated as a unit for data changes. Has an **aggregate root** that controls access. | `Order` (root) + `OrderItems` |
| **Value Object** | Defined by its attributes, not identity. Immutable. Two with same values are equal. | `Money(100, "USD")`, `Address` |
| **Bounded Context** | A boundary within which a domain model is consistent. Same term can mean different things in different contexts. | "Customer" in Sales vs "Customer" in Shipping |
| **Entity** | Defined by identity, not attributes. Mutable over time. | `User` (same user even if name changes) |

**Key rule**: only access entities inside an aggregate through the aggregate root. External code should never hold a reference to an internal entity.

</details>

---

### 15. What is the difference between RabbitMQ and Kafka? When would you use each?

<details>
<summary>Reveal answer</summary>

| Aspect | RabbitMQ | Kafka |
|--------|----------|-------|
| Model | Message broker (push) | Event log (pull) |
| Message retention | Deleted after consumption | Retained for a configurable period |
| Ordering | Per queue | Per partition |
| Throughput | Moderate (thousands/sec) | Very high (millions/sec) |
| Replay | Not supported (message is gone) | Supported (consumers can rewind) |
| Use case | Task queues, RPC, routing | Event streaming, event sourcing, analytics |

**Choose RabbitMQ** when you need traditional messaging: work queues, routing, request-reply.

**Choose Kafka** when you need: event replay, high throughput, multiple consumers reading the same events, event sourcing.

Deep dive: [Messaging](../05-architecture-and-patterns/10-messaging.md)

</details>

---

### 16. What is the Decorator pattern and how is it used in .NET?

<details>
<summary>Reveal answer</summary>

The Decorator pattern **wraps** an object to add behavior dynamically without modifying the original class. All decorators implement the same interface as the wrapped object.

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(int id);
}

// Base implementation
public class OrderRepository : IOrderRepository { ... }

// Decorator — adds caching
public class CachedOrderRepository : IOrderRepository
{
    private readonly IOrderRepository _inner;
    private readonly IMemoryCache _cache;

    public CachedOrderRepository(IOrderRepository inner, IMemoryCache cache)
    {
        _inner = inner;
        _cache = cache;
    }

    public async Task<Order?> GetByIdAsync(int id) =>
        await _cache.GetOrCreateAsync($"order-{id}",
            _ => _inner.GetByIdAsync(id));
}
```

Common in .NET: `HttpClient` handlers (delegating handlers), Polly resilience wrappers, logging decorators, caching decorators.

Deep dive: [Design Patterns](../05-architecture-and-patterns/02-design-patterns.md)

</details>

---

### 17. What is the Mediator pattern and how does MediatR implement it?

<details>
<summary>Reveal answer</summary>

The Mediator pattern reduces **direct dependencies between objects** by having them communicate through a mediator. Instead of A calling B directly, A sends a message to the mediator, which routes it to B.

**MediatR** in .NET:

```csharp
// Command
public record CreateOrderCommand(string Product, int Qty) : IRequest<int>;

// Handler
public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, int>
{
    public async Task<int> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        // create order, return ID
    }
}

// Controller — no direct dependency on handler
app.MapPost("/orders", async (CreateOrderCommand cmd, IMediator mediator) =>
    await mediator.Send(cmd));
```

MediatR also supports **notifications** (one-to-many) and **pipeline behaviors** (cross-cutting concerns like validation, logging).

Deep dive: [Design Patterns](../05-architecture-and-patterns/02-design-patterns.md)

</details>

---

[Back to index](README.md)
