# Structured Logging

**Structured logging** means every log line is a set of named fields (key-value pairs), not a formatted string. The whole point is to make logs queryable by tools (*"show me all logs where `OrderId = 42` in the last 15 min"*) without regex.

In .NET the idiomatic stack is `Microsoft.Extensions.Logging` (abstraction) + a provider (Serilog, OpenTelemetry Logs, Console, etc.). You write against `ILogger<T>`; the provider decides the output format.

## Message templates — the central idea

The placeholder syntax `{Name}` is not string interpolation. The logging framework captures each argument as a named property and passes it to the provider, which can then emit JSON with those properties as first-class fields.

```csharp
// Correct — "UserId" and "OrderTotal" are captured as structured fields
_logger.LogInformation("Order {OrderId} confirmed for user {UserId}, total {OrderTotal}",
    orderId, userId, total);

// WRONG — string interpolation formats eagerly, loses structure
_logger.LogInformation($"Order {orderId} confirmed");
```

The placeholder **names** matter (they become field names); their **order** must match the argument order — the framework aligns by position, not by name. There is a Roslyn analyzer (CA2254 *"Template should be a static expression"*) that flags interpolated templates for exactly this reason.

## Log levels — the exact numeric scale

`Microsoft.Extensions.Logging.LogLevel` has six enabled levels plus `None`:

| Level | Value | Use it for |
|---|---|---|
| `Trace` | 0 | Hot-path debugging. Disabled in production; may contain sensitive data. |
| `Debug` | 1 | Dev/diagnostic detail. |
| `Information` | 2 | Normal flow — request handled, job completed. |
| `Warning` | 3 | Unexpected but recoverable. |
| `Error` | 4 | A failure in the current operation/request. |
| `Critical` | 5 | System-wide failure (data loss, out of disk). |
| `None` | 6 | Suppress everything. |

Serilog's enum (`LogEventLevel`) uses slightly different names: `Verbose`, `Debug`, `Information`, `Warning`, `Error`, `Fatal`. When Serilog is wired as a `Microsoft.Extensions.Logging` provider, `Trace` maps to `Verbose` and `Critical` maps to `Fatal`.

> **Default level** is `Information` if configuration does not specify one. `Trace` and `Debug` are typically off in production.

## Setup — the non-trivial app shape

```csharp
// Program.cs — typical ASP.NET Core setup with Serilog
using Serilog;

Log.Logger = new LoggerConfiguration()
    .MinimumLevel.Information()
    .Enrich.FromLogContext()
    .Enrich.WithProperty("service.name", "orders-api")
    .WriteTo.Console(new Serilog.Formatting.Json.JsonFormatter())
    .CreateLogger();

try
{
    var builder = WebApplication.CreateBuilder(args);
    builder.Host.UseSerilog(); // plugs Serilog into ILogger<T>

    // ... register services
    var app = builder.Build();
    app.Run();
}
finally
{
    Log.CloseAndFlush();
}
```

Key points:
- `UseSerilog()` replaces the default logger factory. Your code keeps using `ILogger<T>` — no Serilog-specific API in business code.
- `Enrich.FromLogContext()` lets scopes (see below) attach properties to every log in a region.
- `JsonFormatter` emits JSON so the log aggregator (Elastic, Loki, CloudWatch, Datadog) can index fields.

## Scopes — attach context to every log in a region

`BeginScope` returns `IDisposable` and adds properties to every log written inside the `using` block. Useful for `CorrelationId`, `TenantId`, `OrderId` — anything you'd otherwise have to repeat in every call.

```csharp
using (_logger.BeginScope(new Dictionary<string, object>
{
    ["OrderId"] = order.Id,
    ["TenantId"] = tenant.Id
}))
{
    _logger.LogInformation("Validating order");
    await _validator.ValidateAsync(order);
    _logger.LogInformation("Order validated"); // both logs carry OrderId + TenantId
}
```

Gotchas from the official docs:
- Not every provider supports scopes. Supported built-ins: `Console`, `AzureAppServicesFile`/`Blob`, Application Insights. Enable via `"IncludeScopes": true` in the provider config.
- Scopes do not flow across `Task.Run` unless the async context itself flows (the default does; `ExecutionContext.SuppressFlow` breaks it).

## Source-generated logging — the fast path

Ad-hoc `LogInformation("… {X}", x)` boxes value types and allocates a `params object?[]`. For hot paths, use the `[LoggerMessage]` attribute. The source generator writes a zero-allocation method for you.

```csharp
public partial class OrderService
{
    private readonly ILogger<OrderService> _logger;

    [LoggerMessage(Level = LogLevel.Information,
        Message = "Order {OrderId} confirmed for user {UserId}")]
    private partial void LogOrderConfirmed(long orderId, long userId);

    public void Confirm(Order o, User u)
    {
        // …
        LogOrderConfirmed(o.Id, u.Id);
    }
}
```

Benefits per the docs: better performance, stronger typing, no stray string constants. Use it when you care about either throughput or compile-time checks.

## Sampling

High-volume services cannot log every request at `Information`. Options:

1. **Category-based filtering** — log `Information` from your code, `Warning` from framework categories. This is the common baseline, configured in `appsettings.json`.
2. **Head-based sampling** — decide at request entry whether to log the whole request (boolean on the scope). Paired with trace sampling so the kept traces and logs match.
3. **Tail-based sampling** — log everything to a fast buffer, keep only the errors/slow ones. Typically implemented outside the app (OTel Collector tail sampling processor).

> **Do not** throttle error logs. Warnings and infos, yes.

## PII redaction

A senior failure mode: a "structured log" serializes a full `Customer` object, which contains email, DOB, address. GDPR/CCPA exposure.

Three layers:

1. **Don't log it.** Scrub at the boundary. `@Customer` with Serilog serializes the full object — use plain placeholders for the fields you actually need.

    ```csharp
    // Leaks
    _logger.LogInformation("Customer {@Customer} placed order", customer);
    // Safe
    _logger.LogInformation("Customer {CustomerId} placed order {OrderId}", customer.Id, order.Id);
    ```

2. **Destructure override.** In Serilog you can register `Destructure.ByTransforming<Customer>(c => new { c.Id, c.Country })` so even `{@Customer}` never emits the PII fields.
3. **Pipeline redaction.** `Microsoft.Extensions.Compliance.Redaction` (shipped as part of the .NET platform extensions for telemetry) classifies data and redacts at log time via `[PrivateData]` / custom taxonomy attributes. Useful when you cannot trust every caller.

## What to log

- Request entry/exit with `TraceId` (the ASP.NET Core request logging middleware does this).
- Domain events: `OrderPlaced`, `PaymentCaptured` — with IDs, not full payloads.
- Exceptions with the exception object as the first argument: `LogError(ex, "Failed to charge {OrderId}", orderId)` — the exception is logged separately from the message template, so stack traces are preserved.
- Slow operations with duration.

## What NOT to log

- Secrets, tokens, passwords, full card numbers, full auth headers.
- Entire request/response bodies by default (PII + cost).
- High-cardinality payloads (raw SQL with parameters, full JSON blobs) at `Information` — demote to `Debug`.

---

[← Previous: The Three Pillars](01-three-pillars.md) | [Next: Metrics →](03-metrics.md) | [Back to index](README.md)
