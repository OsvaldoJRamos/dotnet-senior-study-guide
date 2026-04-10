# SOLID

SOLID is an acronym for five object-oriented design principles that help create more maintainable, flexible, and scalable software.

## S - Single Responsibility Principle (SRP)

**A class should have only one reason to change.**

Each class should have a single responsibility. If a class does too many things, it becomes difficult to maintain.

```csharp
// WRONG - class with multiple responsibilities
public class Order
{
    public void CalculateTotal() { }
    public void SaveToDatabase() { }
    public void SendEmail() { }
}

// CORRECT - each class with one responsibility
public class Order
{
    public decimal CalculateTotal() { ... }
}

public class OrderRepository
{
    public void Save(Order order) { ... }
}

public class NotificationService
{
    public void SendConfirmation(Order order) { ... }
}
```

## O - Open/Closed Principle (OCP)

**Open for extension, closed for modification.**

You should be able to add new behaviors without changing existing code.

```csharp
// WRONG - need to modify the class for each new type
public class DiscountCalculator
{
    public decimal Calculate(string type, decimal value)
    {
        if (type == "VIP") return value * 0.2m;
        if (type == "Premium") return value * 0.1m;
        return 0;
    }
}

// CORRECT - extensible via new implementations
public interface IDiscount
{
    decimal Calculate(decimal value);
}

public class VipDiscount : IDiscount
{
    public decimal Calculate(decimal value) => value * 0.2m;
}

public class PremiumDiscount : IDiscount
{
    public decimal Calculate(decimal value) => value * 0.1m;
}
```

## L - Liskov Substitution Principle (LSP)

**Subclasses should be able to replace their base classes without breaking the program.**

If class B inherits from A, then B should work anywhere that A works.

```csharp
// WRONG - Penguin inherits from Bird but doesn't fly
public class Bird
{
    public virtual void Fly() { Console.WriteLine("Flying..."); }
}

public class Penguin : Bird
{
    public override void Fly() { throw new Exception("Can't fly!"); } // violates LSP
}

// CORRECT - separate contracts
public interface IBird { }
public interface IFlyingBird : IBird
{
    void Fly();
}

public class Eagle : IFlyingBird
{
    public void Fly() { Console.WriteLine("Flying..."); }
}

public class Penguin : IBird { } // does not implement Fly
```

## I - Interface Segregation Principle (ISP)

**No client should be forced to depend on methods it does not use.**

Prefer several small, specific interfaces over one large, generic interface.

```csharp
// WRONG - interface too large
public interface IWorker
{
    void Work();
    void Eat();
    void Sleep();
}

// CORRECT - segregated interfaces
public interface IWorker
{
    void Work();
}

public interface ILivingBeing
{
    void Eat();
    void Sleep();
}
```

## D - Dependency Inversion Principle (DIP)

**Depend on abstractions, not on implementations.**

High-level modules should not depend on low-level modules. Both should depend on abstractions.

```csharp
// WRONG - depends on concrete implementation
public class OrderService
{
    private readonly SqlOrderRepository _repo = new SqlOrderRepository();
}

// CORRECT - depends on abstraction
public class OrderService
{
    private readonly IOrderRepository _repo;

    public OrderService(IOrderRepository repo)
    {
        _repo = repo;
    }
}
```

> DIP is the foundation for Dependency Injection (DI). See more in [Dependency Injection](../08-aspnet-core/01-dependency-injection.md).

---

[Back to index](README.md) | [Next: Design Patterns →](02-design-patterns.md)
