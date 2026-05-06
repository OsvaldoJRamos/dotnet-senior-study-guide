# Domain-Driven Design (DDD)

DDD is an approach to software design where the **domain model** — the code that expresses the business rules — is the center of the system. Everything else (persistence, UI, messaging) is an outer adapter. It's the tool you reach for when the business is the hard part, not the technology.

> Martin Fowler: *"Domain-Driven Design (DDD) is an approach to software development that centers the development on programming a domain model that has a rich understanding of the processes and rules of a domain."* (`martinfowler.com/bliki/DomainDrivenDesign.html`)

DDD comes from Eric Evans' 2003 book *Domain-Driven Design: Tackling Complexity in the Heart of Software*. The vocabulary below is the standard one you'll be asked about.

## When DDD is worth it

| Worth it | Not worth it |
|---|---|
| Complex business rules with real invariants | CRUD app with forms → tables |
| Evolving domain, frequent change | Short-lived internal tool |
| Multiple teams working on one product | One developer, one module |
| Cost of a wrong transition is high (money, compliance) | Cost of a bug is "user reloads the page" |

DDD has overhead — ubiquitous language, modeling sessions, explicit aggregates, repositories. Don't pay it unless the domain pays you back.

## Strategic design

Strategic design is about **carving the problem**: what are the distinct parts of the business, how do they relate, and which ones deserve the most modeling effort.

### Ubiquitous Language

> *"A Ubiquitous Language that embeds domain terminology into the software systems that we build."* (Fowler)

One shared vocabulary across business experts, product, and code. If the business says *"order"* but the code says `PurchaseTicket`, you pay translation cost on every conversation — and introduce bugs at the translation boundary. The names in your code should match the words on the whiteboard.

### Bounded Context

A **bounded context** is the explicit scope within which a particular model applies. The word *Customer* means something different to Sales (a lead with pipeline stage) than to Billing (an invoicing account) than to Support (a person with tickets). Forcing one `Customer` class to satisfy all three is how you get a God Class.

Inside a bounded context:
- The ubiquitous language is consistent.
- The model is internally coherent (no contradictions).
- The team owns the code and the data.

Across bounded contexts:
- Models translate explicitly at the boundary (anti-corruption layers, published contracts, events).

### Context Mapping

How bounded contexts relate. The common relationship types:

| Relationship | Meaning |
|---|---|
| **Partnership** | Two teams succeed or fail together; coordinated planning. |
| **Shared Kernel** | A small shared codebase/schema — rare, high discipline required. |
| **Customer–Supplier** | Downstream depends on upstream; upstream has to honor downstream's needs. |
| **Conformist** | Downstream accepts the upstream model as-is (legacy integration). |
| **Anti-Corruption Layer (ACL)** | Downstream translates upstream's model into its own — used when upstream is messy or unstable. |
| **Open Host Service / Published Language** | Upstream exposes a well-documented contract (REST API, event schema) any consumer can use. |
| **Separate Ways** | Contexts do not integrate — some duplication is cheaper than coupling. |

### Event Storming

A collaborative modeling workshop (created by Alberto Brandolini) where domain experts and developers map a business process using **orange sticky notes for domain events** in chronological order, then layer commands (blue), aggregates (yellow), policies (purple), read models (green), external systems (pink), and hotspots/questions (red).

Used to:
1. Surface the actual process (usually differs from the org chart).
2. Discover bounded context boundaries.
3. Find the aggregates and the invariants they protect.
4. Expose ambiguities early, before any code is written.

## Tactical design

Tactical design is the **building blocks inside a bounded context**. These are the patterns you use in code.

### Entity

An object identified by a stable identity, not by its attribute values. Two `Order` objects with the same fields are *not* the same order — the `OrderId` is what makes them equal.

```csharp
public sealed class Order
{
    public OrderId Id { get; }
    // ... equality by Id
}
```

### Value Object

An object defined entirely by its attributes, **immutable**, and replaceable. Two `Money(100, "BRL")` instances are interchangeable.

```csharp
public sealed record Money(decimal Amount, string Currency)
{
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Currency mismatch");
        return this with { Amount = Amount + other.Amount };
    }
}
```

Value objects cut primitive obsession, centralize validation, and catch entire bug classes at compile time.

### Aggregate

A cluster of entities and value objects treated as a single unit for data changes. Each aggregate has a **root** — the only entity external code references. Invariants that span the cluster are enforced inside the aggregate.

```csharp
public sealed class Order   // aggregate root
{
    private readonly List<OrderLine> _lines = new();
    public OrderId Id { get; }
    public OrderStatus Status { get; private set; }
    public Money Total => Money.Sum(_lines.Select(l => l.Subtotal));

    public void AddLine(ProductId product, int quantity, Money unitPrice)
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOperationException("Cannot modify a confirmed order.");
        _lines.Add(new OrderLine(product, quantity, unitPrice));
    }

    public void Confirm()
    {
        if (!_lines.Any())
            throw new InvalidOperationException("Empty order cannot be confirmed.");
        Status = OrderStatus.Confirmed;
    }
}
```

**Aggregate rules:**

- Exactly one root per aggregate. External code holds references only to the root.
- Transactions are per-aggregate. If two aggregates must change atomically, you have a **design smell** — probably the boundary is wrong, or the change belongs in a domain event + eventual consistency.
- Keep aggregates **small**. Giant aggregates cause lock contention and merge conflicts.

### Domain Service

Behavior that doesn't naturally belong to any single entity or value object — usually because it operates on several at once (e.g., `TransferMoney(fromAccount, toAccount, amount)`). Named using domain language; not a dumping ground for "stuff we didn't want in entities".

### Repository

A collection-like abstraction over aggregate persistence. The domain code talks to `IOrderRepository.GetById(OrderId)`; the infrastructure implements it with EF Core, Dapper, Mongo, whatever.

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(OrderId id, CancellationToken ct);
    Task AddAsync(Order order, CancellationToken ct);
    Task UpdateAsync(Order order, CancellationToken ct);
}
```

> **One repository per aggregate**, not per table. You persist an `Order` including its lines — not an `OrderLine` separately.

### Domain Event

An immutable fact that something important happened in the domain (`OrderConfirmed`, `PaymentFailed`, `AccountClosed`). Raised from inside the aggregate; published after the transaction commits.

```csharp
public record OrderConfirmed(OrderId OrderId, DateTimeOffset At);
```

Domain events enable:
- **Decoupling**: other modules react without the aggregate knowing about them.
- **Eventual consistency** across aggregates/contexts (see [Saga](06-saga-pattern.md)).
- **Audit** and **event sourcing** (see [Event Sourcing](15-event-sourcing.md)).

### Factory

When creating an aggregate is non-trivial (multi-step, needs coordination, not a simple `new`), wrap it in a factory that returns a valid instance.

## Layering in a DDD-style project

A common project layout in a .NET DDD solution:

```
Domain           ← entities, value objects, aggregates, domain events, domain services — NO framework dependencies
Application      ← use cases / command handlers / query handlers, orchestrate domain + repositories
Infrastructure   ← EF Core DbContext, repository implementations, message publishers, external APIs
Presentation     ← ASP.NET Core controllers / minimal APIs / gRPC / CLI
```

The arrow direction in Clean Architecture: dependencies point **inward** to Domain. Domain doesn't know about EF Core, HTTP, Kafka — infrastructure implements interfaces defined in Domain or Application.

See [Clean Architecture](07-clean-architecture.md) for how the layers fit together.

## DDD and CQRS

DDD models are naturally write-centric — aggregates protect invariants. Reads that span aggregates (dashboards, lists, reports) are often better served by a dedicated **read model** — hence DDD pairing with CQRS. See [CQRS](08-cqrs.md) and [Event Sourcing](15-event-sourcing.md).

## Pitfalls

- **DDD on a CRUD app.** All the pattern, none of the payoff. Use vanilla EF Core with anemic entities and move on.
- **God aggregates.** A single `Order` aggregate that owns customers, products, and payments. Split until each aggregate protects a single consistent chunk.
- **"Tactical DDD" without strategic DDD.** Using entities, value objects, and repositories inside a badly-drawn monolith gives you none of the team-scaling benefits.
- **Ubiquitous language inconsistency.** Business says *"shipment"*, code says `Delivery`, DB says `parcel`. Every translation is a bug waiting to happen.
- **Leaking the persistence model into the domain.** Adding `virtual` everywhere for lazy loading, navigation properties for joins, `[Column]` attributes. The domain should be persistable but not shaped by the ORM.
- **Over-eventing.** Emitting a domain event for every property change. Events are business-meaningful ("OrderConfirmed"), not data-change-meaningful ("OrderTotalChanged").

## Rule of thumb

> Strategic DDD answers *"where does this change live?"*. Tactical DDD answers *"how do I keep this change consistent?"*. If your app has no hard invariants and no bounded contexts worth drawing, neither is worth paying for.

---

[← Previous: Anti-Patterns](12-anti-patterns.md) | [Back to index](README.md) | [Next: API Gateway and BFF →](14-api-gateway-and-bff.md)
