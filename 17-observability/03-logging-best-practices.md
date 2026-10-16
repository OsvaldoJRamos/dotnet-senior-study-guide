# Logging Best Practices

[Structured Logging](02-structured-logging.md) covers the *mechanics*: `ILogger<T>`, message templates, scopes, providers, PII redaction. This file covers the *judgment* that separates useful logs from expensive noise:

- which level to pick
- what to log once and where
- what logging costs
- when something should be a **metric**, a **trace**, or an **audit record** instead of a log

## Choosing the level

The official .NET descriptions:

| Level | Official description | Production default |
|---|---|---|
| `Trace` | *"Contain the most detailed messages. These messages might contain sensitive app data. These messages are disabled by default and should **not** be enabled in production."* | Off |
| `Debug` | *"For debugging and development. Use with caution in production due to the high volume."* | Off, or on for one category while investigating |
| `Information` | *"Tracks the general flow of the app. Might have long-term value."* | On for your code |
| `Warning` | *"For abnormal or unexpected events. Typically includes errors or conditions that don't cause the app to fail."* | On |
| `Error` | *"For errors and exceptions that can't be handled. These messages indicate a failure in the current operation or request, not an app-wide failure."* | On |
| `Critical` | *"For failures that require immediate attention. Examples: data loss scenarios, out of disk space."* | On |

The ASP.NET Core docs add a cost warning that includes `Information`: *"Logging at the Trace, Debug, or Information levels produces a high-volume of detailed log messages. To control costs and not exceed data storage limits, log at these levels to a high-volume, low-cost data store."*

### Decision guide by scenario

| Scenario | Level | Why |
|---|---|---|
| Request handled, job finished, order placed | `Information` | Normal flow |
| Client sent invalid input: 400 / 404 / 422 from **your** API | `Information` (or `Debug`) | The caller's mistake, not your failure. OpenTelemetry agrees: *"For HTTP status codes in the 4xx range span status MUST be left unset in case of `SpanKind.SERVER`"* |
| Authentication / authorization failure | `Warning` | Security-relevant. OWASP lists *"Authentication successes and failures"* and *"Authorization (access control) failures"* as events to log. A spike signals an attack. |
| Transient fault, retry attempt N | `Warning` | Azure guidance: *"Log transient faults as warning entries rather than as error entries so that monitoring systems don't detect them as application errors that might trigger false alerts."* |
| Retries exhausted, request failed | `Error` | *"a failure in the current operation or request"* |
| Your API returns 5xx | `Error` | OTel: for 5xx, *"span status SHOULD be set to `Error`"* |
| **Your** outbound call got a 4xx (e.g. 400 from the payment provider) | `Error` (usually) | As a client, a 4xx usually means *your* bug. OTel: 4xx *"SHOULD be set to `Error` in case of `SpanKind.CLIENT`"* |
| Expected, handled exception (concurrency conflict → 409, validation exception) | `Information` / `Warning` | It was handled. `Error` is for what *"can't be handled"*. |
| Missing required configuration at startup, app can't run | `Critical` | Requires immediate attention |
| Disk full, data loss | `Critical` | The docs' own examples |
| SQL text, payloads, loop iterations | `Debug` / `Trace` | Volume and sensitive data |

> **Rule of thumb:** an `Error` should mean that someone should look at this. If an "error" fires 10,000 times a day for an expected condition, people learn to ignore `Error`, and they will miss the real one.

### Level names across ecosystems

| `Microsoft.Extensions.Logging` | Serilog | OpenTelemetry `SeverityNumber` |
|---|---|---|
| `Trace` | `Verbose` | TRACE (1–4) |
| `Debug` | `Debug` | DEBUG (5–8) |
| `Information` | `Information` | INFO (9–12) |
| `Warning` | `Warning` | WARN (13–16) |
| `Error` | `Error` | ERROR (17–20) |
| `Critical` | `Fatal` | FATAL (21–24) |
| `None` | — (no equivalent level) | — |

## Configuring levels: how filter rules are chosen

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "MyCompany.Payments": "Debug"
    },
    "Console": {
      "LogLevel": {
        "Default": "Warning"
      }
    }
  }
}
```

The `Default` / `Microsoft.AspNetCore` pair is what the ASP.NET Core templates generate. Selection rules, from the .NET docs:

1. *"Select all rules that match the provider or its alias. If no match is found, select all rules with an empty provider."*
2. *"From the result of the preceding step, select rules with longest matching category prefix."*
3. *"If multiple rules are selected, take the **last** one."*

Practical consequences:

- **Provider-specific settings win.** *"Settings in `Logging.{ProviderName}.LogLevel` override settings in `Logging.LogLevel`."* In the JSON above, the console only shows `Warning`+, even for `MyCompany.Payments`.
- **Longest prefix wins.** `MyCompany.Payments.Gateway` inherits `Debug`; everything else under `MyCompany` gets `Information`.
- **No configuration means `Information`:** *"If the default log level isn't set, the default log level value is `Information`."*
- *"We recommend setting the minimum default log level in configuration, not in C# code."* Configuration can change per environment without a rebuild.

> With Serilog, its README recommends removing the `"Logging"` section and configuring levels through Serilog itself (`MinimumLevel.Override("Microsoft.AspNetCore", LogEventLevel.Warning)`). Two filter systems in one app is a classic "why is my log missing?" bug.

## Anti-patterns

### 1. Log-and-throw

```csharp
// Anti-pattern: logged here, logged again by every caller up the stack, then by the global handler
catch (HttpRequestException ex)
{
    _logger.LogError(ex, "Payment failed for {OrderId}", orderId);
    throw;
}
```

Sonar rule S2139 (*"Exceptions should be either logged or rethrown but not both"*) explains the cost: *"the upstream code may not be aware that the exception has already been logged. As a result, the same exception gets logged multiple times"*. That gives one incident, five stack traces, and inflated error counts.

Do one or the other:

```csharp
// Option A — handle it here: log + recover
catch (HttpRequestException ex)
{
    _logger.LogWarning(ex, "Payment provider unavailable for {OrderId}, queued for retry", orderId);
    await _retryQueue.EnqueueAsync(orderId, ct);
}

// Option B — can't handle it: add context and rethrow, but DON'T log
catch (HttpRequestException ex)
{
    throw new PaymentFailedException($"Payment failed for order {orderId}", ex);
}
```

The single log entry for unhandled exceptions is written at the boundary (below).

### 2. Exception details in the message

```csharp
// Wrong — stack trace lost, message unique per exception (breaks grouping)
_logger.LogError("Failed: " + ex.Message);
_logger.LogError("Failed: {Error}", ex.ToString());

// Right — exception is its own field; the template stays constant
_logger.LogError(ex, "Failed to charge order {OrderId}", orderId);
```

The OpenTelemetry .NET docs: *"You should avoid adding exception details into the message template."*

### 3. String interpolation

*"The string is evaluated even if the corresponding `LogLevel` isn't enabled"*, and the structure is lost (see [Structured Logging](02-structured-logging.md#message-templates--the-central-idea)). Don't wait for the framework to fix it. A request for interpolated-string `ILogger` overloads (dotnet/runtime #111283) was closed as not planned.

### 4. Logging inside hot loops

10,000 items × one `Information` line each = 10,000 log entries per batch. Log **one summary** (`Processed {Count} items in {ElapsedMs} ms, {Failed} failed`), keep per-item details at `Debug`, and count failures with a metric.

### 5. `Error` for expected conditions

Validation failures, 404s and concurrency conflicts handled with a 409 are not errors. See the decision guide above.

### 6. Counting or alerting by grepping logs

That is a metric's job. See [When a log should be a metric](#when-a-log-should-be-a-metric).

### 7. Log injection

OWASP: *"Perform sanitization on all event data to prevent log injection attacks e.g. carriage return (CR), line feed (LF) and delimiter characters"*. A user-supplied value containing `\n` can forge fake log lines in plain-text sinks. A JSON formatter escapes control characters inside field values, which is one more reason to emit structured JSON rather than text.

## Log once, at the boundary

Let unhandled exceptions bubble up to one place that logs them once and returns a clean `500`. `UseExceptionHandler` *"Catches and logs unhandled exceptions."* Per the source, it logs at `Error` with the event name `UnhandledException`, under the category `Microsoft.AspNetCore.Diagnostics.ExceptionHandlerMiddleware`. Requests aborted by the client are logged at `Debug` instead.

The catch is **`IExceptionHandler`**, and the behavior changed in .NET 10:

| Version | `IExceptionHandler.TryHandleAsync` returns `true` and your handler also logs |
|---|---|
| .NET 8 / 9 | **Logged twice.** The ASP.NET Core 9 docs: *"If the handler does its own logging, the exception is recorded twice: once under the `Microsoft.AspNetCore.Diagnostics.ExceptionHandlerMiddleware` category and once under the handler's category."* (See also dotnet/aspnetcore #54554.) |
| .NET 10+ | **Logged once, by you.** *"if IExceptionHandler.TryHandleAsync returns `true`, then exception diagnostics are no longer recorded by default."* This also suppresses the `error.type` tag on the `http.server.request.duration` metric. |

```csharp
public sealed class GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext, Exception exception, CancellationToken cancellationToken)
    {
        // On .NET 10+ this is the ONLY log entry for the exception — don't skip it
        logger.LogError(exception, "Unhandled exception on {Method} {Path}",
            httpContext.Request.Method, httpContext.Request.Path);

        httpContext.Response.StatusCode = StatusCodes.Status500InternalServerError;
        await httpContext.Response.WriteAsJsonAsync(new { title = "Unexpected error" }, cancellationToken);
        return true;
    }
}

builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails(); // required: parameterless UseExceptionHandler() throws at startup without a fallback
app.UseExceptionHandler();
```

To get the middleware's own diagnostics back on .NET 10, set `ExceptionHandlerOptions.SuppressDiagnosticsCallback = context => false`. On .NET 8/9 the docs' workaround for double logging is to set the `Microsoft.AspNetCore.Diagnostics.ExceptionHandlerMiddleware` category to `None`. That is safe only if your handler logs **every** exception it sees.

> **Upgrade trap:** a .NET 8 app whose handler returned `true` *without* logging (relying on the middleware) silently loses its exception logs **and** the `error.type` metric tag after moving to .NET 10.

## Event IDs

*"An event ID associates a set of events."* *"Some logging providers store the event ID in a field, which allows for filtering on the ID."* Message text changes when someone fixes a typo; an event ID shouldn't. Stable IDs make dashboards, alert queries and runbooks survive refactors.

```csharp
public static partial class PaymentLog
{
    [LoggerMessage(EventId = 2001, Level = LogLevel.Warning,
        Message = "Payment provider {Provider} timed out after {ElapsedMs} ms (attempt {Attempt})")]
    public static partial void ProviderTimeout(this ILogger logger, string provider, long elapsedMs, int attempt);

    [LoggerMessage(EventId = 2002, Level = LogLevel.Error,
        Message = "Payment for order {OrderId} failed after {Attempts} attempts")]
    public static partial void PaymentFailed(this ILogger logger, Exception exception, Guid orderId, int attempts);
}
```

Grouping the messages in one static class also gives you a catalog of everything the module can log.

## Performance: what logging costs

- **Source-generated `[LoggerMessage]`** is the fast path: *"This approach provides the best performance by eliminating boxing, temporary allocations, and message template parsing at runtime."*
- **`IsEnabled` guards: the guidance is nuanced.** Microsoft says a guard avoids *"value-type boxing and an allocation of `object[]`"* for ad-hoc calls. The OpenTelemetry .NET docs (using a source-generated example) note that *"The logging API is highly optimized for the scenario where most loggers are **disabled** for certain log levels"*, and that *"Making an extra call to `IsEnabled` before logging will not give you any performance gain"*. Guard only when **computing an argument** is expensive:

```csharp
if (logger.IsEnabled(LogLevel.Debug))
{
    logger.LogDebug("Cart snapshot {Cart}", JsonSerializer.Serialize(cart)); // expensive argument
}
```

Analyzers that enforce the rules:

| Rule | Title | Enabled by default (.NET 10) |
|---|---|---|
| CA2017 | Parameter count mismatch | Yes, as warning |
| CA2254 | Template should be a static expression | As suggestion |
| CA1848 | Use the LoggerMessage delegates | No: turn it on for hot paths |
| CA1727 | Use PascalCase for named placeholders | No |

## Controlling volume and cost

In order of effort:

1. **Levels and category filters** (above). This is free and usually the biggest win.
2. **Log sampling** (`Microsoft.Extensions.Telemetry`). Keep a fraction of the logs, or only the logs whose trace was sampled:

    ```csharp
    builder.Logging.AddTraceBasedSampler(); // logs follow the trace sampling decision
    ```

    *"Only one sampler can be used at a time. If you register multiple samplers, the last one is used."* The docs recommend sampling `Information`, considering it for `Warning`, and **not** sampling `Error`/`Critical`.

3. **Log buffering** (.NET 9+). Hold low-level logs in memory and emit them **only if something goes wrong**. The .NET blog's motivation: *"You want detailed logs when things go wrong, but you don't want to pay storage costs for logs that never get examined."* Global buffering lives in `Microsoft.Extensions.Telemetry`; per-request buffering lives in `Microsoft.AspNetCore.Diagnostics.Middleware`. You inject `GlobalLogBuffer` / `PerRequestLogBuffer` and call `Flush()` when you hit a failure. Caveats from the docs: *"If the buffer is full, the oldest logs are dropped and never emitted"* and *"The order of logs is not guaranteed to be preserved."* The sampling and buffering APIs were marked stable in `dotnet/extensions` 9.7.0.

4. **Pipeline-level sampling** (OTel Collector tail sampling). See [Structured Logging](02-structured-logging.md#sampling).

## HTTP logging: useful, expensive, risky

```csharp
// HttpLoggingFields.Duration and CombineLogs require .NET 8+
builder.Services.AddHttpLogging(options =>
{
    options.LoggingFields = HttpLoggingFields.RequestPropertiesAndHeaders
                          | HttpLoggingFields.ResponsePropertiesAndHeaders
                          | HttpLoggingFields.Duration;
    options.CombineLogs = true; // one entry per request instead of several
});

app.UseHttpLogging();
```

- The docs carry two warnings: *"HTTP logging **can reduce the performance of an app**, especially when logging the request and response bodies"*, and *"HTTP logging can potentially log personally identifiable information (PII)."*
- Defaults: `LoggingFields` is request + response properties and headers. Headers not on the allow-list are logged as `[Redacted]`. `RequestBodyLogLimit` is 32 KB. `CombineLogs` defaults to `false`.
- With the template's `"Microsoft.AspNetCore": "Warning"`, **nothing shows up** until you enable the category: `"Microsoft.AspNetCore.HttpLogging.HttpLoggingMiddleware": "Information"`.
- Use `IHttpLoggingInterceptor` (`AddHttpLoggingInterceptor<T>()`) to change what is logged per request, e.g. skip health checks or redact a field.
- The **W3CLogger** writes access logs in W3C format to files. *"By default, fields that could contain PII aren't logged."*

> Serilog users usually prefer `UseSerilogRequestLogging()`, which its README says *"condenses these into a single event"* per request.

## Log vs metric vs trace vs audit record

[The Three Pillars](01-three-pillars.md) compares logs, metrics and traces. In practice there is a fourth record type that people wrongly put in logs: the **audit record**.

| | Log | Metric | Trace | Audit record |
|---|---|---|---|---|
| OTel definition / purpose | *"a timestamped text record, either structured (recommended) or unstructured, with optional metadata"* | *"a measurement of a service captured at runtime"* | *"the big picture of what happens when a request is made"* | Durable proof of **who** did **what** to **which** data, **when** |
| Question it answers | Why did *this* event happen? | How many / how fast / how often, in aggregate? | Where did the time go across services? | Who changed X, from what to what? |
| Can it be sampled or dropped? | Yes for `Debug`/`Information`. Never errors. | Pre-aggregated, not per event | Yes (head/tail sampling) | **Never** |
| Cardinality | High is fine | **Low only** | High is fine | High |
| Typical retention | Days to weeks | Months to years (cheap) | Days | Years (legal/compliance) |
| Alert on it? | No: derive a metric | **Yes** | No | No (security events may feed a SIEM) |
| .NET API | `ILogger` | `System.Diagnostics.Metrics` | `Activity` / OpenTelemetry | Your own store (DB table, append-only log) |

### When a log should be a metric

If you find yourself *counting* log lines, you want a counter. The Google SRE Workbook: *"We still recommend metrics-based alerting in such cases: you can increment a counter metric when a particular event happens, and configure an alert based on that metric's value."* And the other side: *"We tend to use logs to find the root cause of an issue, as the information we need is often not available as a metric."*

Emit both, each for its own job:

```csharp
_paymentFailures.Add(1, new KeyValuePair<string, object?>("provider", provider)); // alert on this
logger.PaymentFailed(exception, orderId, attempts);                                // investigate with this
```

The order ID goes in the log. In the metric it would explode cardinality (see [Metrics](04-metrics.md)).

### Why audit is not "a log at Information"

Everything in this file **reduces** logs: level filters, sampling, buffers that *"drop"* entries, rotation, short retention. An audit trail can tolerate none of that. OWASP keeps them apart: *"Process monitoring, audit, and transaction logs/trails etc. are usually collected for different purposes than security event logging, and this often means they should be kept separate."* It also recommends: *"Build in tamper detection so you know if a record has been modified or deleted."*

My recommendation: write audit records to a durable store **in the same transaction** as the change they describe, for example an EF Core `SaveChanges` interceptor writing an `AuditEntries` table, or an outbox. Don't write them through `ILogger`.

## What never to log

OWASP says these *"should usually not be recorded directly in the logs, but instead should be removed, masked, sanitized, hashed, or encrypted"*:

- Session IDs, *"Access tokens"*, *"Authentication passwords"*
- *"Database connection strings"*, *"Encryption keys and other primary secrets"*
- Payment card data, PII, and data the user opted out of collecting

Redaction techniques (`Destructure`, `Microsoft.Extensions.Compliance.Redaction`) are in [Structured Logging](02-structured-logging.md#pii-redaction).

## Checklist

- [ ] Levels follow the decision guide: 4xx from your API is not `Error`, retries are `Warning`, exhausted retries are `Error`
- [ ] Each exception logged **once**, at the handling point or the global boundary, never log-and-rethrow
- [ ] On .NET 10+, `IExceptionHandler` implementations that return `true` log the exception themselves
- [ ] Exceptions passed as the `exception` argument, never formatted into the message
- [ ] Levels configured per category in configuration; one filter system (MEL **or** Serilog)
- [ ] Hot-path logs use `[LoggerMessage]` with stable `EventId`s
- [ ] Anything you count or alert on is also a metric
- [ ] Audit records go to a durable store, not `ILogger`
- [ ] No secrets, tokens, or connection strings in logs; user input can't forge log lines

---

[← Previous: Structured Logging](02-structured-logging.md) | [Next: Metrics →](04-metrics.md) | [Back to index](README.md)
