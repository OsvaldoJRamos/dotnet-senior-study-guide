# Service Lifetimes: Singleton vs Scoped vs Transient

## Summary

| Lifetime | Instance per... | Reuse? | Best for... | Example |
|---|---|---|---|---|
| **Singleton** | Entire application lifetime | Always | Shared and stateless services | Logger, Configuration |
| **Scoped** | HTTP request (web apps) | Per request | Request-specific services | Database context |
| **Transient** | Every injection/request | Never | Lightweight and stateless logic | Email service, Validators |

## Singleton

Classes that will be created **only once** during the entire lifetime of the application. This means less memory usage but also means that whatever they do, they must be **thread safe**.

Logging is a good example of a singleton because the internal implementation, even when writing to files, ensures that logs are written in order and no race condition occurs. An in-memory cache is another good example because it stores state that you want to share across the entire application, but the implementation must ensure thread safety.

Another example is the implementation of a factory pattern. Although it could be used as scoped or transient, since it only creates instances of a given class, making the implementation thread safe and singleton will save application memory.

```csharp
builder.Services.AddSingleton<ILogger, FileLogger>();
builder.Services.AddSingleton<IMemoryCache, MemoryCache>();
```

## Scoped

Classes that are shared **during a scope** — usually a web request or a user action in a desktop application. They are perfect for implementing the **Unit of Work** pattern, where you want to share state during the duration of that operation.

**Database connections and ORM contexts** are good examples because you want to keep all database operations within the same transaction, so all your services within that API request or action should share the same database access.

It cannot be singleton because parallel operations (like different web requests) could affect each other's state (you don't want a rollback from one HTTP request to revert another unrelated one). But it also cannot be transient — if ServiceA calls ServiceB, they would receive different connections and changes would not be shared for that operation.

```csharp
builder.Services.AddScoped<AppDbContext>();
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
```

## Transient

Classes that will be created **every time** they are requested. Perfect for short-lived and stateless classes, such as services that implement business logic.

```csharp
builder.Services.AddTransient<IEmailService, SmtpEmailService>();
builder.Services.AddTransient<IValidator<Pedido>, PedidoValidator>();
```

## Important rule: Captive Dependencies

Never inject a service with a **shorter lifetime** into a service with a **longer lifetime**:

```csharp
// WRONG - Scoped inside Singleton = Captive Dependency
builder.Services.AddSingleton<MeuSingleton>();  // vive pra sempre
builder.Services.AddScoped<MeuScoped>();         // deveria morrer por request

public class MeuSingleton
{
    // MeuScoped vai ficar preso no Singleton e nunca ser descartado!
    public MeuSingleton(MeuScoped scoped) { }
}
```

**Rule:** Singleton > Scoped > Transient (you can only inject a lifetime equal to or longer).

---

[← Previous: Dependency Injection](01-dependency-injection.md) | [Back to index](README.md) | [Next: OAuth 2.0 →](03-oauth2.md)
