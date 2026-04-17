# Metrics

Metrics are **numerical measurements reported over time**. They are what you alert on and dashboard on. In modern .NET (6+) the official instrumentation API is `System.Diagnostics.Metrics` — the same API OpenTelemetry .NET wraps for export.

## The instruments

`System.Diagnostics.Metrics.Meter` is the factory. From a `Meter` you create instruments via `CreateX<T>()` methods. All instruments are thread-safe and allocation-free for up to three tags.

| Instrument | Sync/Async | Semantics | Typical use |
|---|---|---|---|
| `Counter<T>` | sync | Monotonically increasing total. Caller calls `Add`. | Requests served, bytes sent. |
| `UpDownCounter<T>` | sync | Can go up or down. Caller calls `Add` (positive or negative). | Queue depth, active connections. |
| `Histogram<T>` | sync | Distribution of values. Caller calls `Record`. | Request latency, payload size. |
| `Gauge<T>` | sync | Current value. Caller calls `Record`. | Point-in-time value you push. |
| `ObservableCounter<T>` | async | Monotonic total; app holds the value, callback reports it. | Process CPU time. |
| `ObservableUpDownCounter<T>` | async | Up/down; callback reports current total. | Cached items currently held. |
| `ObservableGauge<T>` | async | Callback reports current value. | Pool size, free memory. |

> `UpDownCounter`, `ObservableUpDownCounter`, and `Gauge` are newer — `UpDownCounter` and `ObservableUpDownCounter` shipped with `System.Diagnostics.DiagnosticSource` v7. `Gauge` (synchronous) is in v9 of that package. If you target older runtimes without updating the NuGet, use `ObservableGauge` as a substitute (per the official docs).

`T` must be one of: `byte`, `short`, `int`, `long`, `float`, `double`, `decimal` — pick the smallest that fits to save memory when tags create many series.

## The canonical pattern

```csharp
using System.Diagnostics.Metrics;

public class OrdersMetrics
{
    public const string MeterName = "MyCompany.Orders";

    private readonly Counter<long> _ordersPlaced;
    private readonly Histogram<double> _orderProcessingSeconds;
    private readonly UpDownCounter<int> _pendingOrders;

    public OrdersMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create(MeterName);

        _ordersPlaced = meter.CreateCounter<long>(
            name: "orders.placed",
            unit: "{orders}",
            description: "Number of orders placed.");

        _orderProcessingSeconds = meter.CreateHistogram<double>(
            name: "orders.processing.duration",
            unit: "s",
            description: "Time to process an order.");

        _pendingOrders = meter.CreateUpDownCounter<int>(
            name: "orders.pending",
            unit: "{orders}",
            description: "Orders in the pending queue.");
    }

    public void RecordOrderPlaced(string country) =>
        _ordersPlaced.Add(1, new KeyValuePair<string, object?>("country", country));

    public void RecordProcessingTime(double seconds) =>
        _orderProcessingSeconds.Record(seconds);

    public void Enqueued() => _pendingOrders.Add(1);
    public void Dequeued() => _pendingOrders.Add(-1);
}

// Program.cs
builder.Services.AddSingleton<OrdersMetrics>();
// IMeterFactory is automatically registered by the generic host / WebApplication since .NET 8.
// You can register it manually in other hosts with builder.Services.AddMetrics().
```

Why `IMeterFactory` instead of `new Meter("…")`? Test isolation, lifetime management, and DI. `IMeterFactory` keeps meters with the same name in different service containers distinct — so parallel tests don't observe each other's measurements.

## Naming and units

Per the official .NET docs, follow OpenTelemetry semantic conventions:
- Lowercase, dotted, hierarchical. `http.server.request.duration`, not `HttpServerRequestDuration`.
- Units follow **UCUM**: `s` for seconds, `By` for bytes, `{requests}` for dimensionless counts.
- Record durations in **seconds as a double**, not milliseconds as an int — all modern collectors expect that.

## Tags and cardinality

Tags (OpenTelemetry calls them *attributes*) categorize measurements. Each unique combination of tag values is a separate time series.

```csharp
_ordersPlaced.Add(1,
    new KeyValuePair<string, object?>("country", "BR"),
    new KeyValuePair<string, object?>("payment.method", "card"));
```

Performance notes from the docs:
- `Add` and `Record` are allocation-free for **up to three individually-specified tags**. Beyond that, use `TagList` to stay allocation-free.
- Tool implementations typically become painful above **~1 000 unique tag combinations per instrument**; Histograms are far more expensive (10–100×).
- Never put unbounded-cardinality values (`user_id`, `trace_id`, `order_id`) in tags.

## RED and USE — what to measure

Two canonical methods, both worth naming in an interview:

**RED** (Tom Wilkes, for request-driven services):
- **R**ate — requests per second
- **E**rrors — failing requests per second
- **D**uration — latency distribution

**USE** (Brendan Gregg, for resources like CPU, disks, pools):
- **U**tilization — % busy
- **S**aturation — queued/pending work
- **E**rrors — error events

Google SRE's "four golden signals" (*Monitoring Distributed Systems*, SRE book) are closely related: **Latency, Traffic, Errors, Saturation**. An error that returns fast still counts as a failure — measure successful and failed latency separately.

A good baseline dashboard covers RED for every user-facing endpoint and USE for every bottleneck resource (DB pool, thread pool, disk queue).

## Histograms: the bucket question

A `Histogram` is rendered as a distribution. How that distribution is computed depends on the exporter:
- **OpenTelemetry default** is explicit bucket boundaries `[0, 5, 10, 25, 50, 75, 100, 250, 500, 750, 1000, 2500, 5000, 7500, 10000]`. Those defaults are tuned for milliseconds; if you record seconds for sub-second calls, everything lands in the `[0, 5]` bucket and percentiles are meaningless.
- Since `System.Diagnostics.DiagnosticSource` 9.0.0, you can ship **instrument advice** from the library: `new InstrumentAdvice<double> { HistogramBucketBoundaries = [0.01, 0.05, 0.1, 0.5, 1, 5] }` passed to `CreateHistogram`. OpenTelemetry .NET SDK 1.10.0+ honors that advice.

```csharp
_orderProcessingSeconds = meter.CreateHistogram<double>(
    name: "orders.processing.duration",
    unit: "s",
    description: "Time to process an order.",
    advice: new InstrumentAdvice<double>
    {
        HistogramBucketBoundaries = [0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5]
    });
```

## Prometheus exposition format

The Prometheus text-based format (`Content-Type: text/plain; version=0.0.4`) is the de-facto pull-model wire format. Supported types: `counter`, `gauge`, `histogram`, `summary`, `untyped`.

```
# HELP orders_placed_total Number of orders placed.
# TYPE orders_placed_total counter
orders_placed_total{country="BR",payment_method="card"} 1423
orders_placed_total{country="US",payment_method="pix"} 87

# HELP orders_processing_duration_seconds Time to process an order.
# TYPE orders_processing_duration_seconds histogram
orders_processing_duration_seconds_bucket{le="0.1"} 9801
orders_processing_duration_seconds_bucket{le="0.5"} 9992
orders_processing_duration_seconds_bucket{le="+Inf"} 10000
orders_processing_duration_seconds_sum 412.3
orders_processing_duration_seconds_count 10000
```

Every line: a metric name, optional labels, a float value, optional timestamp. `HELP` and `TYPE` lines describe the metric. Label values are UTF-8 with required escaping for `\`, `"`, and newline.

To serve this from an ASP.NET Core app with OpenTelemetry, add `OpenTelemetry.Exporter.Prometheus.AspNetCore` and register `UseOpenTelemetryPrometheusScrapingEndpoint()`. OTLP-to-Prometheus is the more common modern path — app pushes OTLP to a Collector, Collector exposes Prometheus format.

## Pull vs push

| Model | How | When |
|---|---|---|
| **Pull** (Prometheus) | Prometheus scrapes `/metrics` every 15 s. | Kubernetes, service-discovery-friendly, simple infra. |
| **Push** (OTLP, StatsD, legacy) | App sends metrics to an endpoint. | Short-lived jobs (Lambdas, batch jobs) where pull would miss the process; or behind a Collector that does the push. |

Modern stack: instrument with `System.Diagnostics.Metrics` → export over **OTLP** to a Collector → Collector forwards to whatever backend (Prometheus, Azure Monitor, Datadog, Tempo+Mimir). OTLP is the portability layer; everything else is vendor-specific.

## What NOT to do

- Don't record latencies in milliseconds as `int`. Record seconds as `double`.
- Don't invent your own metric naming scheme when OTel semantic conventions exist for HTTP, DB, messaging, FaaS — they're what backends auto-dashboard against.
- Don't mix `Counter` and `UpDownCounter` semantics ("I'll just decrement my Counter") — collectors will treat a monotonic counter that goes down as a reset and miscalculate rates.
- Don't expose `/metrics` publicly. Bind it to the internal interface or protect it with auth.

---

[← Previous: Structured Logging](02-structured-logging.md) | [Next: Distributed Tracing →](04-distributed-tracing.md) | [Back to index](README.md)
