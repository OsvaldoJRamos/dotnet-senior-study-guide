# Observability

> Read the questions, think about your answer, then click to reveal.

---

### 1. What are the three pillars of observability, and what question is each one best at answering?

<details>
<summary>Reveal answer</summary>

- **Logs** — timestamped structured records of discrete events. Answer *"what exactly happened in this one request?"*
- **Metrics** — pre-aggregated numeric time series. Answer *"how is the system behaving in aggregate right now?"*
- **Traces** — a DAG of spans representing one request crossing services. Answer *"where did time go, and who called whom?"*

The drill-down pattern is **metrics → traces → logs**. Alert on metrics (cheap at high resolution), drill to traces to find the slow path, open logs for the specific failing event. Exemplars on Prometheus histograms let you jump straight from a metric bucket to a sample trace.

Some argue for a fourth pillar: **wide structured events** (Honeycomb's model). Know the debate; the pragmatic senior answer is *"three signals plus exemplars to move between them."*

Deep dive: [The Three Pillars](../14-observability/01-three-pillars.md)

</details>

---

### 2. Why is cardinality the central cost concern in metrics, and where does high-cardinality data belong instead?

<details>
<summary>Reveal answer</summary>

Each unique combination of metric label values produces a separate time series. The .NET docs call out that tag combinations above ~1 000 per instrument already require tool-side filtering, and Histograms are 10–100× more memory-hungry than counters.

**High-cardinality values — user IDs, order IDs, trace IDs, email, request IDs — do not belong on metric labels.** They belong:
- On **logs** (structured fields, cheap to store, queryable).
- On **trace spans** (attributes, one per request, sampled).
- As **exemplars** on a metric sample (a pointer to a trace ID, not a label).

The interview signal: you don't "solve" cardinality by sampling the metric harder — you move the high-cardinality dimension to a different signal.

Deep dive: [The Three Pillars](../14-observability/01-three-pillars.md)

</details>

---

### 3. Why is `logger.LogInformation($"Order {id}")` wrong, and what is the correct form?

<details>
<summary>Reveal answer</summary>

String interpolation formats eagerly and throws away the structure — the logging provider only sees the final formatted string. Structured logging relies on **message templates**, where `{PropertyName}` placeholders are captured as named fields and shipped to the provider as key-value pairs.

Correct:

```csharp
_logger.LogInformation("Order {OrderId} confirmed for user {UserId}", orderId, userId);
```

Now a JSON-formatting provider emits `{"OrderId": 42, "UserId": 7, ...}`, queryable by field. The analyzer **CA2254 — *"Template should be a static expression"*** flags the interpolation mistake. Placeholder order matters; names do not align arguments — position does.

Also know: `LogError(ex, "…")` passes the exception separately from the template so the stack trace is preserved. And for hot paths, use `[LoggerMessage]` source generation for zero-allocation logging.

Deep dive: [Structured Logging](../14-observability/02-structured-logging.md)

</details>

---

### 4. What is a log scope, and when would you use one?

<details>
<summary>Reveal answer</summary>

A scope is an `IDisposable` returned by `ILogger.BeginScope(...)` that attaches the same set of properties to **every** log written inside the `using` block. Lives until disposed. Flows across `await` because it's on `ExecutionContext`.

Use it to avoid repeating context in every log call:

```csharp
using (_logger.BeginScope(new Dictionary<string, object>
{
    ["OrderId"] = order.Id,
    ["TenantId"] = tenant.Id
}))
{
    _logger.LogInformation("Validating order");
    await _validator.ValidateAsync(order);
    _logger.LogInformation("Validated");   // both logs carry OrderId, TenantId
}
```

Gotchas:
- Not every provider supports scopes. The Console, Azure App Services, and Application Insights providers do; enable with `"IncludeScopes": true`.
- `Task.Run` preserves scopes by default; `ExecutionContext.SuppressFlow` breaks them.

Deep dive: [Structured Logging](../14-observability/02-structured-logging.md)

</details>

---

### 5. Walk through the `System.Diagnostics.Metrics` instruments. When would you use each?

<details>
<summary>Reveal answer</summary>

All created from a `Meter`. Sync vs async refers to whether the app calls `Add`/`Record` directly or provides a callback.

| Instrument | Sync/Async | Use for |
|---|---|---|
| `Counter<T>` | sync | Monotonic total (`Add` only, non-negative). Requests served, bytes sent. |
| `UpDownCounter<T>` | sync | Value that goes up and down (`Add` with +/-). Queue depth, active connections. |
| `Histogram<T>` | sync | Distribution (`Record`). Request latency, payload size. |
| `Gauge<T>` | sync | Current value (`Record`). Synchronous push of a single latest value. (`System.Diagnostics.DiagnosticSource` v9+) |
| `ObservableCounter<T>` | async | Monotonic total read via callback. CPU time, total bytes since start. |
| `ObservableUpDownCounter<T>` | async | Up/down via callback. Current cache entry count. |
| `ObservableGauge<T>` | async | Current value via callback. Pool size, free memory. |

Rules from the docs:
- For counting, prefer `Counter` unless it's easier to read from a variable (then `ObservableCounter`).
- For **timing**, use `Histogram` — averages hide tail latency.
- For bounded things like queue depth and cache size, use `UpDownCounter` or `ObservableUpDownCounter`.
- Record durations as **seconds as `double`**, not milliseconds as `int`.
- `Add`/`Record` are allocation-free for up to 3 tags; use `TagList` beyond that.

Deep dive: [Metrics](../14-observability/03-metrics.md)

</details>

---

### 6. Explain RED, USE, and the Four Golden Signals. How do they relate?

<details>
<summary>Reveal answer</summary>

Three overlapping frameworks for *"what should I actually measure?"*

- **RED** (Tom Wilkes) — for **request-driven services**. **R**ate, **E**rrors, **D**uration.
- **USE** (Brendan Gregg) — for **resources** (CPU, disks, pools). **U**tilization, **S**aturation, **E**rrors.
- **Four Golden Signals** (Google SRE book) — for **user-facing systems**. **L**atency, **T**raffic, **E**rrors, **S**aturation.

Golden Signals ≈ RED + Saturation. Use them together: RED on every user-facing endpoint, USE on every bottleneck resource. The SRE book is explicit about measuring successful-request latency separately from failed-request latency — "a fast 500 is still a failure."

Deep dive: [Metrics](../14-observability/03-metrics.md) and [Alerting and Dashboards](../14-observability/07-alerting-and-dashboards.md)

</details>

---

### 7. Decode this W3C `traceparent` header: `00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01`.

<details>
<summary>Reveal answer</summary>

Four dash-separated fields:

| Field | Value | Meaning |
|---|---|---|
| `version` | `00` | Only valid version today; `ff` is reserved-invalid. |
| `trace-id` | `4bf92f3577b34da6a3ce929d0e0e4736` | 16 bytes. Globally unique per trace. All-zeros forbidden. |
| `parent-id` | `00f067aa0ba902b7` | 8 bytes. The caller's span ID. |
| `trace-flags` | `01` | 8-bit field; low bit is the `sampled` flag. `01` = the caller may have recorded the trace. |

Companion header `tracestate` carries vendor-specific state (up to 32 key=value members) — opaque to apps that don't own the vendor prefix.

Interview gotcha: the `sampled` flag is a **hint**, not a contract. The receiver is free to decide independently. OpenTelemetry's `ParentBasedSampler` is the one that honors it.

Deep dive: [Distributed Tracing](../14-observability/04-distributed-tracing.md)

</details>

---

### 8. In .NET, what is the relationship between `System.Diagnostics.Activity` and an OpenTelemetry Span?

<details>
<summary>Reveal answer</summary>

**They are the same thing.** .NET chose not to introduce a parallel `Span` type — `Activity` **is** the OpenTelemetry span, and `ActivitySource` **is** the tracer.

Consequences:
- You can instrument a library with `System.Diagnostics.DiagnosticSource` only; **no OpenTelemetry dependency required**.
- OpenTelemetry .NET is an *exporter/processor* — it listens to Activities and ships them as OTLP/Jaeger/Zipkin.
- Since .NET 5, `Activity.DefaultIdFormat` is `W3C`, so IDs are emitted in W3C Trace Context format by default.
- Key properties map directly: `TraceId`, `SpanId`, `ParentSpanId`, `Baggage`, `TraceStateString`, `Kind`, `Status`.

Consequence for design: `StartActivity` returns `null` when no listener is subscribed. Always null-check (`activity?.SetTag(...)`) to stay allocation-free on the no-tracing path.

Deep dive: [Distributed Tracing](../14-observability/04-distributed-tracing.md) and [Correlation IDs](../14-observability/05-correlation-ids.md)

</details>

---

### 9. How do you propagate trace context through a message queue (Kafka, RabbitMQ, Service Bus)?

<details>
<summary>Reveal answer</summary>

HTTP headers don't exist on a message. The OpenTelemetry semantic conventions say: **put `traceparent` (and `tracestate`) into the message headers or properties**, same string format as HTTP.

Producer side:

```csharp
using var activity = Source.StartActivity("order.publish", ActivityKind.Producer);
props.Headers["traceparent"] = Activity.Current!.Id;                    // W3C string
props.Headers["tracestate"]  = Activity.Current!.TraceStateString ?? "";
```

Consumer side:

```csharp
var parentId = Encoding.UTF8.GetString((byte[])e.BasicProperties.Headers["traceparent"]);
using var activity = Source.StartActivity("order.consume", ActivityKind.Consumer, parentId: parentId);
```

In practice, the messaging library or its OTel contrib instrumentation (MassTransit, `OpenTelemetry.Instrumentation.ConfluentKafka`, AWS SDK instrumentation) handles this for you — you rarely hand-write it. Use `ActivityKind.Producer` on publish and `ActivityKind.Consumer` on receive so the trace viewer renders queue hops correctly.

Deep dive: [Correlation IDs](../14-observability/05-correlation-ids.md)

</details>

---

### 10. Define SLI, SLO, and SLA. What's the question that separates an SLO from an SLA?

<details>
<summary>Reveal answer</summary>

Quoting the Google SRE book:

- **SLI** — *"a carefully defined quantitative measure of some aspect of the level of service that is provided."* Typically expressed as a ratio of good events to valid events.
- **SLO** — *"a target value or range of values for a service level that is measured by an SLI."* Internal commitment.
- **SLA** — *"an explicit or implicit contract with your users that includes consequences of meeting (or missing) the SLOs they contain."* External contract with teeth.

The disambiguator: **"what happens if we miss it?"** No consequence → SLO. Refund, credit, contract penalty → SLA.

Derived concept: **error budget** — the fraction (`1 − SLO`) you are *allowed* to miss over a window. At 99.9 % that's 0.1 %, or ~43 minutes per 30 days. Budget healthy → ship the risky change. Budget burned → freeze risky changes, prioritize reliability.

Practical rule: internal SLO should be **tighter** than published SLA so normal-sized incidents don't trigger customer-facing penalties.

Deep dive: [SLI, SLO, SLA](../14-observability/06-sli-slo-sla.md)

</details>

---

### 11. What is burn-rate alerting, and why is multi-window multi-burn-rate better than a single threshold?

<details>
<summary>Reveal answer</summary>

**Burn rate** = (fraction of error budget consumed in a window) ÷ (fraction of the SLO window that window covers). For a 99.9 % SLO, burn rate 1× means you'd spend the full 30-day budget in 30 days; burn rate 14.4× means you'd spend it in ~2 days.

A single threshold alert has two failure modes:
- **Too sensitive** → flaps on short spikes that don't threaten the budget.
- **Too slow** → you're already out of budget by the time it fires.

The Google SRE Workbook (*Alerting on SLOs*, Table 5-8) recommends **multi-window multi-burn-rate**: for a 99.9 % SLO,

| Severity | Long window | Short window | Burn rate | Budget at fire |
|---|---|---|---|---|
| Page | 1 h | 5 min | 14.4× | 2 % |
| Page | 6 h | 30 min | 6× | 5 % |
| Ticket | 3 d | 6 h | 1× | 10 % |

Both windows must exceed the threshold. The long window filters noise; the short window (1/12 of the long by default) ensures alerts **reset within minutes of recovery**, not hours — the single biggest cause of on-call fatigue.

Deep dive: [SLI, SLO, SLA](../14-observability/06-sli-slo-sla.md) and [Alerting and Dashboards](../14-observability/07-alerting-and-dashboards.md)

</details>

---

### 12. What's the difference between symptom-based and cause-based alerts? Why does it matter for on-call?

<details>
<summary>Reveal answer</summary>

- **Symptom-based** alerts fire on things users feel — high error rate, p95 latency over budget, stale data, dropped messages.
- **Cause-based** alerts fire on things that *might* cause user pain — CPU high, thread pool queued, disk filling, queue depth growing.

The Google SRE guidance: **page on symptoms, not on causes.** A cause can exist without user impact — auto-scaling handled it, the circuit breaker stopped cascading, traffic was low. Paging on causes at 03:00 produces toil without customer benefit.

Causes still matter — they go on **dashboards** (to aid investigation) and on **tickets** (for trend issues like rising GC time). Never the pager.

Rules of thumb for page quality:
- Every page is **actionable** (if the response is "ack and go back to sleep", delete the alert).
- Every page is **novel** (consolidate duplicates; auto-remediate the known-safe ones).
- Every page is **urgent** (not urgent = ticket).
- Every alert has a **runbook** link in the annotation. An alert with no runbook is a bug.

Deep dive: [Alerting and Dashboards](../14-observability/07-alerting-and-dashboards.md)

</details>

---

[Back to index](README.md)
