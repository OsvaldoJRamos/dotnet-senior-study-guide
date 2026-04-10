# Middleware Pipeline

## What is Middleware

Middleware are components that form an HTTP request **processing pipeline** in ASP.NET Core. Each middleware can:

- Process the request **before** passing it to the next one
- Process the response **after** the next one returns
- **Short-circuit** the pipeline (not call the next one)

```
Request → [Middleware 1] → [Middleware 2] → [Middleware 3] → Endpoint
Response ← [Middleware 1] ← [Middleware 2] ← [Middleware 3] ←
```

## Order matters

The order in which middlewares are registered defines the execution order. **Wrong order = subtle bugs**.

```csharp
var app = builder.Build();

// Recommended order by Microsoft:
app.UseExceptionHandler("/error");   // 1. Catches exceptions
app.UseHsts();                        // 2. HSTS
app.UseHttpsRedirection();            // 3. Redirects HTTP -> HTTPS
app.UseStaticFiles();                 // 4. Static files (short-circuits here)
app.UseRouting();                     // 5. Determines the endpoint
app.UseCors();                        // 6. CORS
app.UseAuthentication();              // 7. Identifies who the user is
app.UseAuthorization();               // 8. Checks if access is allowed
app.MapControllers();                 // 9. Executes the endpoint
```

> If `UseAuthentication` comes **after** `UseAuthorization`, the authorization won't have the user identified = bug.

## Custom middleware

### Inline (simple)

```csharp
app.Use(async (context, next) =>
{
    // Before the next middleware
    var sw = Stopwatch.StartNew();
    
    await next(context);
    
    // After the next middleware
    sw.Stop();
    Console.WriteLine($"{context.Request.Path} took {sw.ElapsedMilliseconds}ms");
});
```

### Class (reusable)

```csharp
public class RequestTimingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<RequestTimingMiddleware> _logger;

    public RequestTimingMiddleware(RequestDelegate next, ILogger<RequestTimingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var sw = Stopwatch.StartNew();
        
        await _next(context);
        
        sw.Stop();
        _logger.LogInformation("{Method} {Path} completed in {Elapsed}ms",
            context.Request.Method, context.Request.Path, sw.ElapsedMilliseconds);
    }
}

// Registration
app.UseMiddleware<RequestTimingMiddleware>();
```

## Map and MapWhen (branching)

Allows creating **branches** in the pipeline for specific routes:

```csharp
// Middleware only for routes starting with /api
app.MapWhen(
    context => context.Request.Path.StartsWithSegments("/api"),
    appBuilder => appBuilder.UseMiddleware<ApiKeyMiddleware>()
);

// Complete branch for /health
app.Map("/health", appBuilder =>
{
    appBuilder.Run(async context =>
    {
        await context.Response.WriteAsync("OK");
    });
});
```

## Short-circuit

Middleware can stop the pipeline without calling `next`:

```csharp
app.Use(async (context, next) =>
{
    if (context.Request.Headers["X-Api-Key"] != "my-key")
    {
        context.Response.StatusCode = 401;
        await context.Response.WriteAsync("Unauthorized");
        return; // does not call next = short-circuit
    }

    await next(context);
});
```

## Middleware vs Filters

| Aspect | Middleware | Filter |
|---------|-----------|--------|
| Scope | **Every** HTTP request | Only requests that reach **MVC/Minimal API** |
| Endpoint access | No endpoint context | Has access to action, model binding, etc. |
| Order | Defined by registration | Defined by order + type (Authorization, Resource, Action, etc.) |
| Ideal use | Cross-cutting (logging, CORS, auth) | MVC-specific logic (validation, action caching) |

## Key points for interviews

1. **Registration order = execution order** — getting the order wrong is a common bug
2. **RequestDelegate** is the delegate that represents the next middleware
3. Middlewares are **singletons** — be careful with Scoped service injection (use `InvokeAsync` with parameters)
4. `app.Run()` is a **terminal** middleware — it doesn't call `next`
5. `app.Use()` can call or not call `next` — it's the middleware's decision

---

[← Previous: API Resilience](04-api-resilience.md) | [Next: Background Services →](06-background-services.md) | [Back to index](README.md)
