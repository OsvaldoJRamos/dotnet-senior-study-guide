# ASP.NET Core

> Read the questions, think about your answer, then click to reveal.

---

### 1. What are the three DI service lifetimes in ASP.NET Core, and when would you choose each?

<details>
<summary>Reveal answer</summary>

- **Singleton** -- one instance for the app's entire lifetime. Use for stateless services, caches, configuration wrappers.
- **Scoped** -- one instance per HTTP request (or per scope). Use for `DbContext`, unit-of-work, per-request state.
- **Transient** -- new instance every time it's requested. Use for lightweight, stateless helpers.

Choose the **narrowest lifetime that still meets your needs** to avoid unintentional shared state.

Deep dive: [Service Lifetimes](../06-aspnet-core/02-service-lifetimes.md)

</details>

---

### 2. What is the "captive dependency" problem, and how does ASP.NET Core help prevent it?

<details>
<summary>Reveal answer</summary>

A **captive dependency** happens when a longer-lived service (e.g., Singleton) depends on a shorter-lived one (e.g., Scoped). The scoped service gets "captured" and effectively becomes a singleton, which can cause stale data or concurrency bugs.

ASP.NET Core provides **scope validation** (enabled by default in Development): it throws an `InvalidOperationException` at startup if a singleton tries to resolve a scoped service.

```csharp
// This would throw at startup in Development
builder.Services.AddSingleton<MySingleton>(); // depends on IMyScoped
builder.Services.AddScoped<IMyScoped, MyScoped>();
```

Deep dive: [Service Lifetimes](../06-aspnet-core/02-service-lifetimes.md)

</details>

---

### 3. Why does the order of middleware registration matter? What happens if you put `UseAuthentication` after `UseAuthorization`?

<details>
<summary>Reveal answer</summary>

Middleware executes in the **order it is registered**. Each middleware can short-circuit the pipeline or pass the request to the next one. If `UseAuthentication()` comes **after** `UseAuthorization()`, the authorization middleware runs before the user identity is established -- so every request appears unauthenticated and authorization checks fail.

Correct order:

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

Deep dive: [Middleware](../06-aspnet-core/05-middleware.md)

</details>

---

### 4. How do you create custom middleware, and when would you short-circuit the pipeline?

<details>
<summary>Reveal answer</summary>

Custom middleware is a class with an `InvokeAsync` method that receives `HttpContext` and a `RequestDelegate next`:

```csharp
public class ApiKeyMiddleware(RequestDelegate next)
{
    public async Task InvokeAsync(HttpContext context)
    {
        if (!context.Request.Headers.ContainsKey("X-Api-Key"))
        {
            context.Response.StatusCode = 401;
            return; // short-circuit -- don't call next
        }
        await next(context);
    }
}
```

**Short-circuiting** is useful for validation, caching, or health-check responses where you want to stop processing early and return immediately.

Deep dive: [Middleware](../06-aspnet-core/05-middleware.md)

</details>

---

### 5. What are the main differences between Minimal APIs and Controllers? When would you pick one over the other?

<details>
<summary>Reveal answer</summary>

| Aspect | Minimal APIs | Controllers |
|--------|-------------|-------------|
| Boilerplate | Very little, lambda-based | More ceremony (classes, attributes) |
| Filters | Endpoint filters | Action filters, model validation |
| API versioning | Manual or via packages | Built-in attribute support |
| Best for | Microservices, small APIs | Large APIs, complex routing |

Choose **Minimal APIs** for simple, fast-to-build services. Choose **Controllers** when you need rich model binding, content negotiation, or a large surface area with shared filters.

Deep dive: [Minimal APIs](../06-aspnet-core/08-minimal-apis.md)

</details>

---

### 6. How do you consume a scoped service (like DbContext) inside an `IHostedService`?

<details>
<summary>Reveal answer</summary>

`IHostedService` is registered as a **singleton**, so you cannot inject scoped services directly (captive dependency). Instead, inject `IServiceScopeFactory` and create a scope manually:

```csharp
public class MyWorker(IServiceScopeFactory scopeFactory) : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        using var scope = scopeFactory.CreateScope();
        var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
        // use db...
    }
}
```

Deep dive: [Background Services](../06-aspnet-core/06-background-services.md)

</details>

---

### 7. What happens if `ExecuteAsync` in a `BackgroundService` throws an unhandled exception?

<details>
<summary>Reveal answer</summary>

In .NET 6+, an unhandled exception in `ExecuteAsync` will **stop the host** by default (the app crashes). In earlier versions, the exception was silently swallowed and the service just stopped running.

You can control this behavior with `HostOptions.BackgroundServiceExceptionBehavior`:
- `StopHost` (default in .NET 6+) -- crash the app.
- `Ignore` -- swallow the exception and stop the service silently.

**Best practice**: always wrap your work in a `try/catch` with proper logging.

Deep dive: [Background Services](../06-aspnet-core/06-background-services.md)

</details>

---

### 8. What is the difference between `IMemoryCache` and `IDistributedCache`? When would you use each?

<details>
<summary>Reveal answer</summary>

- **`IMemoryCache`** -- stores data in the **process memory** of a single server. Fast, but not shared across instances. Good for single-server apps or data that's OK to be duplicated per instance.
- **`IDistributedCache`** -- abstraction over an external store (Redis, SQL Server). Shared across all instances. Necessary for horizontally scaled apps.

Use `IMemoryCache` for hot, per-server lookups. Use `IDistributedCache` when consistency across instances matters (e.g., session state, feature flags).

Deep dive: [Caching](../06-aspnet-core/07-caching.md)

</details>

---

### 9. What is a "cache stampede" and how can you prevent it?

<details>
<summary>Reveal answer</summary>

A **cache stampede** (thundering herd) happens when a popular cache entry expires and many concurrent requests all try to regenerate it at the same time, overloading the data source.

Prevention strategies:
- **Lock / semaphore**: only one thread regenerates; others wait.
- **`GetOrCreateAsync` with `SemaphoreSlim`**: serialize cache population.
- **Stale-while-revalidate**: serve the stale value while refreshing in the background.
- **Randomized expiration**: add jitter to TTLs so entries don't all expire at once.

Deep dive: [Caching](../06-aspnet-core/07-caching.md)

</details>

---

### 10. How does SignalR scale across multiple server instances, and what role does a Redis backplane play?

<details>
<summary>Reveal answer</summary>

By default, SignalR keeps connection state **in-process**, so a message sent on Server A won't reach clients connected to Server B. A **Redis backplane** acts as a pub/sub broker: every server publishes messages to Redis, and every server subscribes to them, ensuring all clients receive broadcasts regardless of which server they're connected to.

```csharp
builder.Services.AddSignalR()
    .AddStackExchangeRedis(connectionString);
```

Deep dive: [SignalR](../06-aspnet-core/09-signalr.md)

</details>

---

### 11. What transport protocols does SignalR support, and how does it negotiate them?

<details>
<summary>Reveal answer</summary>

SignalR supports three transports, tried in order of preference:
1. **WebSockets** -- full-duplex, best performance.
2. **Server-Sent Events (SSE)** -- server-to-client only, uses HTTP.
3. **Long Polling** -- fallback, higher latency.

The client sends a **negotiate** request; the server responds with available transports. The client picks the best one the environment supports. You can force a specific transport if needed.

Deep dive: [SignalR](../06-aspnet-core/09-signalr.md)

</details>

---

### 12. What is the difference between an access token and a refresh token in OAuth 2.0?

<details>
<summary>Reveal answer</summary>

- **Access token** -- short-lived (minutes), sent with every API request in the `Authorization` header. Grants access to resources. If compromised, limited damage due to short TTL.
- **Refresh token** -- long-lived (days/weeks), stored securely on the client. Used to obtain a **new access token** without re-authenticating the user. Should be rotated on use and revocable.

**Never send the refresh token to a resource server** -- it is only sent to the authorization server's token endpoint.

Deep dive: [OAuth 2.0](../06-aspnet-core/03-oauth2.md)

</details>

---

### 13. How does ASP.NET Core validate a JWT? What happens if you forget to set `ValidateIssuer` or `ValidateAudience`?

<details>
<summary>Reveal answer</summary>

ASP.NET Core's `JwtBearerHandler` validates: **signature** (using the issuer's key), **expiration**, **issuer**, and **audience**. If you disable `ValidateIssuer` or `ValidateAudience`, any token from any identity provider or intended for any application would be accepted -- a serious security hole.

```csharp
options.TokenValidationParameters = new TokenValidationParameters
{
    ValidateIssuer = true,   // always keep true
    ValidateAudience = true, // always keep true
    ValidateLifetime = true,
    ValidIssuer = "https://my-idp.com",
    ValidAudience = "my-api"
};
```

Deep dive: [OAuth 2.0](../06-aspnet-core/03-oauth2.md)

</details>

---

### 14. How does the built-in Rate Limiting middleware work in .NET 7+? Name two common algorithms.

<details>
<summary>Reveal answer</summary>

The `Microsoft.AspNetCore.RateLimiting` middleware applies rate limits using a **policy-based model**. You configure policies in `Program.cs` and apply them globally, per-endpoint, or per-controller.

Common algorithms:
- **Fixed Window** -- limits requests in fixed time intervals (e.g., 100 req/min).
- **Sliding Window** -- smoother than fixed; divides the window into segments.
- **Token Bucket** -- allows bursts up to a bucket capacity, refills at a steady rate.
- **Concurrency** -- limits how many requests are processed simultaneously.

Deep dive: [API Resilience](../06-aspnet-core/04-api-resilience.md)

</details>

---

### 15. What is the difference between `IOptions<T>`, `IOptionsSnapshot<T>`, and `IOptionsMonitor<T>`?

<details>
<summary>Reveal answer</summary>

| Interface | Lifetime | Reloads on change? | Use case |
|-----------|----------|-------------------|----------|
| `IOptions<T>` | Singleton | No | Static config, read once at startup |
| `IOptionsSnapshot<T>` | Scoped | Yes, per request | Config that may change; accessed per request |
| `IOptionsMonitor<T>` | Singleton | Yes, via callback | Singleton services that need live config updates |

**Key rule**: if your service is a singleton and needs fresh config, use `IOptionsMonitor<T>`. If scoped, use `IOptionsSnapshot<T>`.

Deep dive: [Dependency Injection](../06-aspnet-core/01-dependency-injection.md)

</details>

---

### 16. Your API calls an external service that occasionally times out. How would you apply retry + circuit breaker using Polly in .NET 8?

<details>
<summary>Reveal answer</summary>

In .NET 8, use `Microsoft.Extensions.Http.Resilience` which integrates Polly v8:

```csharp
builder.Services.AddHttpClient("catalog")
    .AddStandardResilienceHandler(); // retry + circuit breaker + timeout out of the box
```

Or customize:

```csharp
builder.Services.AddHttpClient("catalog")
    .AddResilienceHandler("custom", builder =>
    {
        builder.AddRetry(new HttpRetryStrategyOptions
        {
            MaxRetryAttempts = 3,
            BackoffType = DelayBackoffType.Exponential
        });
        builder.AddCircuitBreaker(new HttpCircuitBreakerStrategyOptions
        {
            SamplingDuration = TimeSpan.FromSeconds(10),
            FailureRatio = 0.5
        });
    });
```

**Order matters**: retry should wrap circuit breaker (retry is outer, circuit breaker is inner).

Deep dive: [API Resilience](../06-aspnet-core/04-api-resilience.md)

</details>

---

### 17. What are health checks in ASP.NET Core, and how do they integrate with container orchestrators?

<details>
<summary>Reveal answer</summary>

Health checks expose HTTP endpoints (e.g., `/healthz`) that report the app's status. You register checks for dependencies (database, Redis, external APIs) and map them:

```csharp
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString)
    .AddRedis(redisConnection);

app.MapHealthChecks("/healthz");
```

Container orchestrators (Kubernetes, Docker) poll these endpoints for **liveness** (is the app alive?) and **readiness** (can it handle traffic?) probes. A failing liveness probe triggers a restart; a failing readiness probe removes the pod from the load balancer.

Deep dive: [API Resilience](../06-aspnet-core/04-api-resilience.md)

</details>

---
