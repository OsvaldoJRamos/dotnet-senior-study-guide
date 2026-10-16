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

Deep dive: [The Three Pillars](../17-observability/01-three-pillars.md)

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

Deep dive: [The Three Pillars](../17-observability/01-three-pillars.md)

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

Deep dive: [Structured Logging](../17-observability/02-structured-logging.md)

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

Deep dive: [Structured Logging](../17-observability/02-structured-logging.md)

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

Deep dive: [Metrics](../17-observability/04-metrics.md)

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

Deep dive: [Metrics](../17-observability/04-metrics.md) and [Alerting and Dashboards](../17-observability/08-alerting-and-dashboards.md)

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

Deep dive: [Distributed Tracing](../17-observability/05-distributed-tracing.md)

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

Deep dive: [Distributed Tracing](../17-observability/05-distributed-tracing.md) and [Correlation IDs](../17-observability/06-correlation-ids.md)

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

Deep dive: [Correlation IDs](../17-observability/06-correlation-ids.md)

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

Deep dive: [SLI, SLO, SLA](../17-observability/07-sli-slo-sla.md)

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

Deep dive: [SLI, SLO, SLA](../17-observability/07-sli-slo-sla.md) and [Alerting and Dashboards](../17-observability/08-alerting-and-dashboards.md)

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

Deep dive: [Alerting and Dashboards](../17-observability/08-alerting-and-dashboards.md)

</details>

---

### 13. What are the three pillars of observability, and what does each answer?

<details>
<summary>Reveal answer</summary>

| Pillar | Format | Answers |
|--------|--------|---------|
| **Logs** | Timestamped records of discrete events | *What happened at this moment?* |
| **Metrics** | Aggregated numeric time series | *How often / how much / how long?* |
| **Traces** | Causally linked spans across services | *Where did time go for this request?* |

They're complementary, not substitutes:
- **Logs** are great for forensic deep-dives but expensive at high cardinality.
- **Metrics** are cheap, real-time, and dashboardable but lose per-request detail.
- **Traces** show the shape of a distributed call chain but sample a tiny fraction in production.

Modern stacks unify them under **OpenTelemetry** — one SDK emitting all three, one exporter, one correlation model (TraceId / SpanId).

Deep dive: [Three Pillars](../17-observability/01-three-pillars.md)

</details>

---

### 14. What is the difference between SLI, SLO, and SLA?

<details>
<summary>Reveal answer</summary>

- **SLI (Service Level Indicator)** — a **measurement** of service behavior. E.g., "% of requests under 300 ms," "successful requests / total requests." Pick a small set that actually reflects user experience.
- **SLO (Service Level Objective)** — an **internal target** on an SLI. E.g., "99.9% of requests under 300 ms over a rolling 28 days." Drives engineering priorities and the **error budget** (100% − SLO).
- **SLA (Service Level Agreement)** — an **external, contractual** promise with consequences (credits, fines) if you miss. SLA targets are **always laxer** than SLOs, so you have internal headroom before contractual breach.

Hierarchy: SLI ⊆ SLO ⊆ SLA. You measure (SLI), commit internally (SLO), promise externally (SLA).

Deep dive: [SLI / SLO / SLA](../17-observability/07-sli-slo-sla.md)

</details>

---

### 15. Why use a correlation ID, and where should it come from?

<details>
<summary>Reveal answer</summary>

A **correlation ID** (or trace ID) is a single identifier attached to every log line, metric label, and downstream call for one logical request. It lets you reconstruct "what the user saw" across services without guessing.

Where it comes from (in order of preference):
1. **Trust upstream if present** — if the request arrives with `traceparent` (W3C Trace Context) or `X-Correlation-Id`, reuse it. Enables end-to-end correlation across system boundaries.
2. **Generate on first ingress** — at your API gateway or the outermost middleware, generate a GUID / W3C trace ID and propagate.
3. **Propagate on every outbound call** — HTTP headers, message properties (Kafka, RabbitMQ), gRPC metadata.

OpenTelemetry's `Activity` / `ActivitySource` in .NET handle this automatically — the SDK reads and writes `traceparent`. Log `Activity.Current?.TraceId` with every entry (via `ILogger` scopes or a structured logging field).

Deep dive: [Correlation IDs](../17-observability/06-correlation-ids.md)

</details>

---

### 16. Pick the log level: (a) your API returns 404 for an unknown order, (b) a transient DB error triggers retry 1 of 3, (c) all 3 retries fail and the request returns 500, (d) your call to the payment provider returns 400, (e) a concurrency conflict is handled with a 409.

<details>
<summary>Reveal answer</summary>

| Case | Level | Why |
|---|---|---|
| (a) 404 from **your** API | `Information` (or `Debug`) | The caller's mistake, not your failure. OTel leaves span status **unset** for 4xx on server spans. |
| (b) Retry attempt | `Warning` | Azure guidance: log transient faults *"as warning entries rather than as error entries so that monitoring systems don't detect them as application errors that might trigger false alerts."* |
| (c) Retries exhausted, 500 | `Error` | *"a failure in the current operation or request"* |
| (d) 400 **from** a dependency | `Error` (usually) | As the client, a 4xx usually means your bug. OTel says client-span status SHOULD be `Error` for 4xx. |
| (e) Handled conflict → 409 | `Information` / `Warning` | It was handled. `Error` is for what *"can't be handled"*. |

Rule of thumb: `Error` means someone should look at it. Expected conditions logged as `Error` train people to ignore the level.

Deep dive: [Logging Best Practices](../17-observability/03-logging-best-practices.md)

</details>

---

### 17. What's wrong with catching an exception, logging it with `LogError`, and rethrowing it? What should you do instead?

<details>
<summary>Reveal answer</summary>

**Log-and-throw** logs the same exception at every layer that does it, plus once more at the global handler. One incident becomes five stack traces, error counts inflate, and alerts misfire. Sonar rule S2139: *"Exceptions should be either logged or rethrown but not both."*

Instead, do **one** of:
- **Handle it:** log (often `Warning`) and recover (fallback, queue for retry, return a result).
- **Can't handle it:** rethrow **without logging**. Wrap it to add context: `throw new PaymentFailedException($"Payment failed for order {orderId}", ex);`

Unhandled exceptions are then logged **once**, at the boundary (`UseExceptionHandler` / `IExceptionHandler`). Always pass the exception as the `exception` argument (`LogError(ex, "...")`), never formatted into the message.

Deep dive: [Logging Best Practices](../17-observability/03-logging-best-practices.md#anti-patterns)

</details>

---

### 18. Your `IExceptionHandler` returns `true` and doesn't log anything. What happens to the exception's log entry on .NET 8 vs .NET 10?

<details>
<summary>Reveal answer</summary>

- **.NET 8 / 9:** the exception handler middleware still logs it (`Error`, event `UnhandledException`, category `Microsoft.AspNetCore.Diagnostics.ExceptionHandlerMiddleware`). If your handler **also** logs, the exception appears **twice**.
- **.NET 10+:** *"if IExceptionHandler.TryHandleAsync returns `true`, then exception diagnostics are no longer recorded by default."* With a non-logging handler, **the exception is not logged at all**. The `error.type` tag on the `http.server.request.duration` metric is suppressed as well.

Fix: log inside `TryHandleAsync` (the single log entry), or restore the old behavior with `ExceptionHandlerOptions.SuppressDiagnosticsCallback = context => false`. This is an upgrade trap worth checking when moving to .NET 10.

Deep dive: [Logging Best Practices](../17-observability/03-logging-best-practices.md#log-once-at-the-boundary)

</details>

---

### 19. Given this configuration, which `Information` logs from category `MyCompany.Payments.Gateway` reach the console provider?

```json
{
  "Logging": {
    "LogLevel": { "Default": "Information", "MyCompany.Payments": "Debug" },
    "Console": { "LogLevel": { "Default": "Warning" } }
  }
}
```

<details>
<summary>Reveal answer</summary>

**None.** The console only shows `Warning` and above.

Rule selection: first, *"Select all rules that match the provider or its alias"*. The `Console` section matches, so only its rules are candidates. Then the longest matching category prefix wins, and if several match, the last one is taken. The console's only rule is `Default: Warning`. *"Settings in `Logging.{ProviderName}.LogLevel` override settings in `Logging.LogLevel`."*

Other providers (no provider-specific section) use the general rules: `MyCompany.Payments.Gateway` matches the longer prefix `MyCompany.Payments` → `Debug`.

Also: with nothing configured the default is `Information`, and the docs recommend setting levels in configuration rather than code. With Serilog, configure levels in Serilog (`MinimumLevel.Override`), not in both systems.

Deep dive: [Logging Best Practices](../17-observability/03-logging-best-practices.md#configuring-levels-how-filter-rules-are-chosen)

</details>

---

### 20. Your team alerts on "more than 50 `PaymentFailed` log lines in 5 minutes". What's wrong with that, and what would you do instead?

<details>
<summary>Reveal answer</summary>

Counting log lines is a metric's job done badly. Log pipelines sample, filter, buffer and drop entries, add ingestion lag, and are expensive to query. A changed message template silently breaks the alert.

The Google SRE Workbook: *"We still recommend metrics-based alerting in such cases: you can increment a counter metric when a particular event happens, and configure an alert based on that metric's value."*

Emit **both**, each for its own job:
- A **counter** (`payments.failed`, low-cardinality tags like `provider`). Alert on this.
- A **log** with the high-cardinality context (`OrderId`, exception) and a stable `EventId`. Investigate with this.

*"We tend to use logs to find the root cause of an issue, as the information we need is often not available as a metric."*

Deep dive: [Logging Best Practices](../17-observability/03-logging-best-practices.md#when-a-log-should-be-a-metric)

</details>

---

### 21. Compliance asks for "a log of every change to customer bank details". Why shouldn't that go through `ILogger`?

<details>
<summary>Reveal answer</summary>

Logging pipelines are routinely configured to be **lossy**: level filters, sampling, buffers where *"the oldest logs are dropped and never emitted"*, rotation, short retention. An audit trail can tolerate none of that. It needs completeness, long retention and tamper evidence.

OWASP keeps them apart: audit trails *"are usually collected for different purposes than security event logging, and this often means they should be kept separate"*. It also recommends tamper detection.

Better: write an **audit record** to a durable store **in the same transaction** as the change (an `AuditEntries` table via an EF Core `SaveChanges` interceptor, or an outbox). Then the change and its audit entry commit or fail together. Keep the bank details themselves masked, both in the audit record and in any log.

Deep dive: [Logging Best Practices](../17-observability/03-logging-best-practices.md#why-audit-is-not-a-log-at-information)

</details>

---

### 22. The .NET 10 SDK flags `_logger.LogDebug("Cart {Cart}", JsonSerializer.Serialize(cart))` with CA1873. Why, and what are the fixes?

<details>
<summary>Reveal answer</summary>

CA1873 is *"Avoid potentially expensive logging"* (enabled by default in .NET 10 as a suggestion). The docs: *"When logging methods are called, their arguments are evaluated regardless of whether the logging level is enabled."* So the cart is serialized on every call, even when `Debug` is filtered out in production.

Fixes, from the rule page:
- Guard the call with `logger.IsEnabled(LogLevel.Debug)`.
- Use source-generated logging (`[LoggerMessage]`) and pass the object, not a precomputed string.
- Don't compute expensive values in logging arguments unless needed.

Don't wrap **every** log call in `IsEnabled`: for cheap arguments the logging API already short-circuits disabled levels. Guard only when computing an argument is expensive.

Deep dive: [Logging Best Practices](../17-observability/03-logging-best-practices.md#performance-what-logging-costs)

</details>

---

[Back to index](README.md)
