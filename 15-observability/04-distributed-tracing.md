# Distributed Tracing

**Distributed tracing** records the path of a single request as it crosses multiple processes and services, so you can see *where the time went* end-to-end. A trace is a tree of **spans**; a span is one unit of work (an HTTP call, a DB query, a message handler) with a start time, a duration, a status, attributes, and events.

## Span, trace, context

Per the OpenTelemetry specification:

- **Span** — one operation. Has operation name, start/end timestamps, attributes (key-value pairs), events (timestamped sub-entries), links (to related spans, possibly in other traces), and a status.
- **Trace** — a **DAG** of spans connected by parent/child relationships, implicitly defined by its spans. One logical end-to-end operation.
- **Context** — the small immutable bag (`trace_id`, `span_id`, flags, baggage) that flows with the request. *"Context propagation"* means serializing this into a transport (HTTP headers, Kafka headers, gRPC metadata) on the way out and deserializing on the way in.

## W3C Trace Context — what actually flows across HTTP

The propagation format everyone agrees on (OpenTelemetry, Azure Monitor, AWS X-Ray with ADOT, Google Cloud Trace, Jaeger modern versions) is **W3C Trace Context**.

Two headers:

```
traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
tracestate: congo=t61rcWkgMzE,rojo=00f067aa0ba902b7
```

`traceparent` = `version-traceid-parentid-flags`:

| Field | Size | Meaning |
|---|---|---|
| `version` | 2 hex | `00` is the only valid version today. `ff` is reserved-invalid. |
| `trace-id` | 32 hex (16 bytes) | Globally unique trace identifier. All-zeros is forbidden. |
| `parent-id` | 16 hex (8 bytes) | The span ID of the caller. All-zeros is forbidden. |
| `trace-flags` | 2 hex (8 bits) | Low bit is the *sampled* flag. Other bits are reserved. |

The *sampled* bit (value `01`) indicates the caller **may have** recorded the trace. It is a hint — the receiver still makes its own decision.

`tracestate` carries vendor-specific context: up to 32 list-members, key=value pairs. Opaque to anyone who doesn't own the vendor prefix.

## The .NET mapping — `Activity` = OpenTelemetry Span

.NET chose not to introduce a new `Span` type. Instead:
- `System.Diagnostics.Activity` **is** the OpenTelemetry span.
- `System.Diagnostics.ActivitySource` **is** the OpenTelemetry tracer (a factory for Activities).

That means you can instrument your .NET code with **zero OpenTelemetry dependency** — just `System.Diagnostics.DiagnosticSource`. OpenTelemetry's .NET SDK is an *exporter/processor* that listens to Activities. This is a deliberate Microsoft design choice.

Key `Activity` properties (all from the Microsoft docs):

| Property | Meaning |
|---|---|
| `TraceId` | 16-byte W3C trace ID. |
| `SpanId` | 8-byte ID of this activity/span. |
| `ParentSpanId` | Parent's `SpanId`. |
| `ParentId` | String form of the parent ID (W3C `traceparent` or hierarchical). |
| `TraceStateString` | The W3C `tracestate` header value. |
| `Baggage` | Key-value pairs that flow to children. |
| `IdFormat` | `W3C` or `Hierarchical`. .NET 5+ defaults to W3C. |
| `Kind` | `Internal`, `Server`, `Client`, `Producer`, `Consumer`. |
| `Status` | `Unset`, `Ok`, `Error`. |
| `Recorded` | Whether this trace should be recorded (sampled-in). |

> `DefaultIdFormat` was flipped to W3C in .NET 5. If you target older, set `Activity.DefaultIdFormat = ActivityIdFormat.W3C; Activity.ForceDefaultIdFormat = true;` at startup.

## Creating spans

```csharp
using System.Diagnostics;

public class OrderService
{
    // One ActivitySource per library, usually named after the assembly.
    private static readonly ActivitySource Source =
        new("MyCompany.Orders", version: "1.0.0");

    public async Task<Order> PlaceOrderAsync(Cart cart, CancellationToken ct)
    {
        // StartActivity returns null when no listener has opted in — always null-check.
        using var activity = Source.StartActivity("PlaceOrder", ActivityKind.Internal);
        activity?.SetTag("cart.items", cart.Items.Count);
        activity?.SetTag("cart.total", cart.Total);

        try
        {
            var order = await _repo.SaveAsync(cart, ct);
            activity?.SetTag("order.id", order.Id);
            activity?.SetStatus(ActivityStatusCode.Ok);
            return order;
        }
        catch (Exception ex)
        {
            activity?.SetStatus(ActivityStatusCode.Error, ex.Message);
            activity?.AddException(ex);
            throw;
        }
    }
}
```

`StartActivity` returns `null` when **no one is listening**. Checking with `?.` is the allocation-free pattern for non-sampled traces.

## Wiring OpenTelemetry as the exporter

Packages:
- `OpenTelemetry.Extensions.Hosting` — registration helpers for the generic host / `WebApplication`.
- `OpenTelemetry.Exporter.OpenTelemetryProtocol` — OTLP exporter (gRPC or HTTP/protobuf).
- `OpenTelemetry.Instrumentation.AspNetCore` — auto-instrument incoming HTTP.
- `OpenTelemetry.Instrumentation.Http` — auto-instrument outgoing `HttpClient`.

```csharp
using OpenTelemetry.Resources;
using OpenTelemetry.Trace;
using OpenTelemetry.Metrics;

builder.Services.AddOpenTelemetry()
    .ConfigureResource(r => r.AddService(
        serviceName: "orders-api",
        serviceVersion: "1.0.0"))
    .WithTracing(t => t
        .AddSource("MyCompany.Orders")           // your ActivitySources
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter())                      // ships to the collector
    .WithMetrics(m => m
        .AddMeter("MyCompany.Orders")
        .AddAspNetCoreInstrumentation()
        .AddHttpClientInstrumentation()
        .AddOtlpExporter());
```

`AddSource("…")` must match the `ActivitySource` name exactly — otherwise your spans are silently ignored. Same pattern for `AddMeter(…)` on the metrics side.

> Per the OpenTelemetry .NET status page, **Logs, Metrics, and Traces** are all at **stable** status for the SDK and OTLP exporter. It is safe to use in production.

## Exporters — what they actually talk to

| Exporter | Wire format | Common target |
|---|---|---|
| **OTLP** (`OpenTelemetry.Exporter.OpenTelemetryProtocol`) | gRPC or HTTP/protobuf | OpenTelemetry Collector, Azure Monitor, Datadog, New Relic, Grafana Tempo, Honeycomb — almost everything modern accepts OTLP. |
| **Jaeger** (legacy, exporter deprecated by OTel) | Thrift/gRPC | Jaeger backend. Use OTLP → Collector → Jaeger instead. |
| **Zipkin** | JSON over HTTP | Zipkin backend. |
| **Console** | stdout | Local dev. |
| **Prometheus** (metrics only) | Scrape `/metrics` | Prometheus. |

The modern architecture is: **app → OTLP → OpenTelemetry Collector → N vendor backends**. The Collector does batching, retries, sampling, and fan-out, so the app only speaks one protocol.

## Sampling — head vs tail

Spans are expensive at scale. You do not send all of them.

- **Head sampling** — decide at trace root whether to record. Simple (e.g., `TraceIdRatioBasedSampler(0.1)` for 10 %), but dumb — you can't prefer error traces because you don't know the outcome yet.
- **Tail sampling** — buffer the whole trace, then decide. The OTel Collector has a `tail_sampling` processor that can keep 100 % of error traces, 100 % of traces slower than p99, and 1 % of everything else. Recommended for anything non-trivial.

For context propagation to work across a service that doesn't record, the `traceparent` header still flows through — the receiver may decide to record. This is the whole point of separating *propagation* from *sampling*.

## Sampling decision flag — gotcha

The W3C spec is explicit: the `sampled` flag in `traceparent` is a **hint**, not a contract. If you are implementing a multi-vendor stack, the downstream service is free to ignore it. OpenTelemetry's `ParentBasedSampler` is the one that actually respects it.

## What NOT to do

- Don't create one `ActivitySource` per class — one per library/assembly is the convention.
- Don't put high-cardinality data in span attributes you also plan to aggregate; spans are cheap to store but expensive to aggregate. That's metrics territory.
- Don't set span status to `Ok` just because no exception was thrown. A 4xx HTTP response with a business failure is not `Ok` — by the OTel HTTP semconv, client-side treats 4xx as `Error`, server-side leaves `Unset` unless it's a 5xx.
- Don't assume every downstream honors `traceparent`. Test it — old AWS SDK versions, some gRPC libraries on old runtimes, and any service that strips unknown headers can break propagation.

---

[← Previous: Metrics](03-metrics.md) | [Next: Correlation IDs →](05-correlation-ids.md) | [Back to index](README.md)
