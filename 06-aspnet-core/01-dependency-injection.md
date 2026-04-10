# Dependency Injection (DI), IoC, DIP and Service Locator

## DIP (Dependency Inversion Principle)

A SOLID principle that states we should **depend on abstractions, not on implementations**. In simple terms, this means depending on the interface rather than the class. This reduces **coupling** between classes, making testing, maintenance, and reuse easier.

> See more in [SOLID](../05-architecture-and-patterns/01-solid.md)

## IoC (Inversion of Control)

It is the **pattern** that says: "don't create your dependencies, receive them from the outside". Instead of a class creating its own objects, someone else provides them.

## DI (Dependency Injection)

It is the **technique** that implements Inversion of Control. In other words, when we say we are working with Dependency Injection, we are actually applying the Inversion of Control pattern.

Instead of instantiating things inside my class, I externalize that responsibility and inject the dependencies into my class. That is, my class stops being responsible for creating and becomes dependent on an implementation.

```csharp
// WITHOUT DI - class creates its dependency
public class OrderService
{
    private readonly OrderRepository _repo = new OrderRepository();
}

// WITH DI - dependency is injected
public class OrderService
{
    private readonly IOrderRepository _repo;

    public OrderService(IOrderRepository repo)
    {
        _repo = repo; // received from the outside
    }
}
```

## Service Locator

Performs the mapping. Given an abstraction X, it will use implementation Y. ASP.NET Core already comes with this container built-in, although there are other containers that do this such as Ninject.

To configure this we should go to the startup and define it as singleton, scoped, or transient.

```csharp
// Program.cs / Startup.cs
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
builder.Services.AddTransient<IEmailService, SmtpEmailService>();
builder.Services.AddSingleton<ICacheService, RedisCacheService>();
```

## Injection methods

### 1. Constructor Injection (recommended)
```csharp
public class OrderService
{
    private readonly IOrderRepository _repo;
    public OrderService(IOrderRepository repo) => _repo = repo;
}
```

### 2. Method Injection
```csharp
public void Process([FromServices] IOrderRepository repo)
{
    // used in controllers, minimal APIs
}
```

### 3. Primary Constructor (C# 12)
```csharp
public class OrderService(IOrderRepository repo)
{
    public void Create() => repo.Save(new Order());
}
```

---

[Back to index](README.md) | [Next: Service Lifetimes →](02-service-lifetimes.md)
