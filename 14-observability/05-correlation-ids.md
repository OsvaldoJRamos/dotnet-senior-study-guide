# Correlation IDs and Context Propagation

A **correlation ID** is a single identifier that lets you join logs, metrics, and traces belonging to the same logical request across services, threads, queues, and retries. In modern .NET you almost never need to build this yourself — the `Activity` API does it, and the value that flows is the **W3C `traceparent`**.

## What "correlation" actually means

You have three IDs that sound similar but are different:

| ID | Scope | Value |
|---|---|---|
| **`TraceId`** | Whole request across all services | 16 bytes, hex-encoded in `traceparent`. |
| **`SpanId`** | One operation within the trace | 8 bytes. A new span = new `SpanId`, same `TraceId`. |
| **`ParentSpanId`** | Immediate caller's span | Empty at the trace root. |

A "correlation ID" in most modern stacks **is** the `TraceId`. Teams that predate W3C Trace Context still sometimes maintain a separate `X-Correlation-Id` or `X-Request-Id`; when you see both, the pragmatic move is to carry the legacy header for back-compat, but log the W3C `TraceId` as the primary key.

## How it flows in .NET — `Activity` does it for you

`Activity.Current` is an `AsyncLocal<Activity?>` — it **flows across `await`, `Task.Run`, and `ThreadPool` work items** automatically, because .NET's async infrastructure captures `ExecutionContext`.

```csharp
// Start of a request (ASP.NET Core already does this for you)
using var activity = Source.StartActivity("HandleOrder");

await Task.Run(async () =>
{
    // Activity.Current inside here is still the same Activity
    // because ExecutionContext flowed.
    var current = Activity.Current;
    await _inner.DoWorkAsync();
});
```

The only ways to break this flow:
- Calling `ExecutionContext.SuppressFlow()` before a task.
- Starting work via `ThreadPool.UnsafeQueueUserWorkItem`.
- Fire-and-forget `new Thread(...)` without capturing context manually.

If you're doing any of those, you're bypassing tracing by design — re-start an `Activity` with the captured parent context inside.

## HTTP — automatic with `HttpClient` and ASP.NET Core

When you install `OpenTelemetry.Instrumentation.Http` and `OpenTelemetry.Instrumentation.AspNetCore`, propagation is handled for you:

- **Outgoing** `HttpClient` call → `traceparent` and `tracestate` are added to the request headers before sending.
- **Incoming** request in ASP.NET Core → `traceparent` is read and set as the parent of the new `Activity`.

Without OpenTelemetry, since .NET 5 the `HttpClient` handler sets `traceparent` automatically when an `Activity` is active. This is baseline runtime behavior — you get it free.

## Context propagation through messaging

Queues don't propagate HTTP headers. You have to put the context on the message yourself.

### The OpenTelemetry messaging convention

The OTel semantic conventions say: put `traceparent` and `tracestate` into the message properties/headers, same string format as HTTP.

```csharp
// Producer — attach traceparent to outgoing message
public async Task PublishAsync(OrderEvent evt, CancellationToken ct)
{
    using var activity = Source.StartActivity("order.publish", ActivityKind.Producer);
    activity?.SetTag("messaging.system", "rabbitmq");

    var props = _channel.CreateBasicProperties();
    props.Headers = new Dictionary<string, object>();

    // Inject current W3C context into message headers
    if (Activity.Current is not null)
    {
        props.Headers["traceparent"] = Activity.Current.Id; // W3C format string
        if (!string.IsNullOrEmpty(Activity.Current.TraceStateString))
            props.Headers["tracestate"] = Activity.Current.TraceStateString;
    }

    _channel.BasicPublish(exchange: "orders", routingKey: "placed", basicProperties: props,
        body: JsonSerializer.SerializeToUtf8Bytes(evt));
}

// Consumer — restore the context on the handler side
public async Task HandleAsync(BasicDeliverEventArgs e, CancellationToken ct)
{
    var parentId = e.BasicProperties.Headers?.TryGetValue("traceparent", out var v) == true
        ? Encoding.UTF8.GetString((byte[])v!) : null;
    var parentState = e.BasicProperties.Headers?.TryGetValue("tracestate", out var s) == true
        ? Encoding.UTF8.GetString((byte[])s!) : null;

    using var activity = Source.StartActivity("order.consume", ActivityKind.Consumer,
        parentId: parentId!);
    if (parentState is not null) activity!.TraceStateString = parentState;

    // ... handle the message; all logs inside inherit the original TraceId
}
```

In practice, the messaging libraries or their OpenTelemetry contrib instrumentation packages (MassTransit, `OpenTelemetry.Instrumentation.AWS`, `OpenTelemetry.Instrumentation.ConfluentKafka`, etc.) do this for you — you rarely write the header code by hand.

## Baggage — context that is NOT a trace ID

**Baggage** is a separate W3C spec (`baggage` header). It propagates application-level key-value pairs alongside the trace, intended for things like `tenant_id`, `experiment_id`, or `user_country` that you want available in every downstream span.

```
baggage: userId=alice,serverNode=DF%2028,isProduction=false
```

Constraints per the W3C spec:
- Lowercase header name. UTF-8 values, percent-encoded for non-ASCII.
- Up to **64 list-members** and **8192 bytes** per header must be propagated.
- Keys are ASCII tokens (per RFC 7230).

In .NET, baggage is accessed via `Activity.Baggage` / `Activity.AddBaggage`:

```csharp
Activity.Current?.AddBaggage("tenant_id", tenant.Id.ToString());

// …later, possibly in another service after propagation…
var tenantId = Activity.Current?.GetBaggageItem("tenant_id");
```

> **Do not** put sensitive data in baggage. It flows as a plain header to every service downstream, including third parties, by default. The W3C spec is explicit that baggage should not contain sensitive information.

Baggage vs `tracestate`:
- **`tracestate`** is for **tracing-system vendors** (their state, opaque to the app).
- **`baggage`** is for **your application**.

## Activity vs ILogger — wiring the trace_id into logs

ASP.NET Core's default logging middleware + OpenTelemetry already add `TraceId` / `SpanId` to log entries via `Activity.Current`. If you're rolling your own logger, capture them yourself:

```csharp
// Serilog enricher — adds trace_id / span_id to every log while an Activity is active
public class ActivityEnricher : ILogEventEnricher
{
    public void Enrich(LogEvent evt, ILogEventPropertyFactory f)
    {
        var a = Activity.Current;
        if (a is null) return;
        evt.AddOrUpdateProperty(f.CreateProperty("trace_id", a.TraceId.ToHexString()));
        evt.AddOrUpdateProperty(f.CreateProperty("span_id", a.SpanId.ToHexString()));
    }
}

Log.Logger = new LoggerConfiguration()
    .Enrich.With<ActivityEnricher>()
    .WriteTo.Console(new JsonFormatter())
    .CreateLogger();
```

Or just call `.Enrich.WithSpan()` from `Serilog.Enrichers.Span` (community) which does the same thing.

## W3C `traceparent` vs legacy hierarchical IDs

Before .NET 5, `Activity.Id` used a hierarchical format (`|rootId.childId.grandchildId`). From .NET 5 onward the default is `ActivityIdFormat.W3C`. If you integrate with a legacy system still emitting hierarchical IDs, set:

```csharp
Activity.DefaultIdFormat = ActivityIdFormat.W3C;
Activity.ForceDefaultIdFormat = true; // override the parent's format
```

`ForceDefaultIdFormat = true` guarantees .NET emits W3C outward even when it received a hierarchical parent.

## What NOT to do

- Don't generate a new GUID as "correlation ID" in every service. Use the `TraceId` from the incoming request.
- Don't log the `SpanId` as the correlation ID. Log both; the join key across services is `TraceId`.
- Don't cross an `async void` boundary and expect context to flow. Use `async Task`.
- Don't propagate PII via baggage. Baggage is world-readable by every downstream service.

---

[← Previous: Distributed Tracing](04-distributed-tracing.md) | [Next: SLI, SLO, SLA →](06-sli-slo-sla.md) | [Back to index](README.md)
