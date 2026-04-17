# The Three Pillars of Observability

**Logs, metrics, and traces** are the three telemetry signals most observability stacks are built on. They are not interchangeable — each answers a different question, has a different cost profile, and has different cardinality constraints.

## The one-liner for each

| Signal | What it is | Best at answering |
|---|---|---|
| **Log** | A timestamped, structured record of a discrete event. | *"What exactly happened in this single request?"* |
| **Metric** | A numeric measurement aggregated over time (counter, gauge, histogram). | *"How is the system behaving in aggregate right now?"* |
| **Trace** | A tree of spans describing one request as it crosses services. | *"Where did time go in this slow request, and who called whom?"* |

Logs and traces are **per-event** — volume scales with request count. Metrics are **pre-aggregated** — volume scales with cardinality of labels, not with request count.

## Cardinality: the cost axis you will be interviewed on

Cardinality = the number of unique label combinations you emit.

- A counter `http_requests_total{route, status}` with 50 routes × 6 status classes = 300 time series. Cheap.
- Add `user_id` as a label → one time series per user. For a system with millions of users, your Prometheus / metrics backend will either drop data or bill you heavily. The .NET docs warn that tag combinations above ~1 000 for a single instrument already require tool-side filtering, and Histograms are typically 10–100× more memory-hungry than other metrics.

The senior rule: **high-cardinality data (user IDs, order IDs, request IDs) belongs in logs or traces, not in metric labels.**

```csharp
// WRONG — user.id explodes cardinality
ordersCounter.Add(1,
    new KeyValuePair<string, object?>("user.id", userId),
    new KeyValuePair<string, object?>("country", country));

// CORRECT — low-cardinality label on the metric; user.id lives on the log/span
ordersCounter.Add(1, new KeyValuePair<string, object?>("country", country));
_logger.LogInformation("Order {OrderId} placed by user {UserId}", orderId, userId);
```

## When to reach for which

**Metric first** when you need to know *something is off*:
- Error rate, p95 latency, queue depth, CPU, RPS. You alert on metrics, not on logs.

**Trace next** when a metric says something is off and you need to know *where*:
- p99 checkout latency regressed → pull a trace from the slow bucket → see the downstream call that's timing out.

**Log last** when you need to know *why* for one specific event:
- A specific `order_id` was rejected → read its structured log with exception + context.

This is the "metrics → traces → logs" drill-down pattern. It minimizes cost: metrics are cheap to keep at high resolution forever; traces and logs can be sampled or kept for a shorter retention.

## What each is NOT for

| Anti-use | Why it breaks |
|---|---|
| Counting events by tailing logs | Non-atomic, expensive at scale, misses events if a log line is dropped. Use a counter. |
| Alerting on raw log patterns | `ERROR` grep alerts are noisy and lag behind metric-based alerts. Convert the condition into a metric, alert on the metric. |
| Storing `trace_id` as a metric label | Unique per request = cardinality explosion. `trace_id` is an **exemplar** attached to a metric sample, not a label. |
| Logging every request body for "observability" | Cost and PII risk. Sample bodies, or put them in traces with sampling. |

## The "fourth pillar" discussion

Some vendors and Honeycomb in particular argue the three pillars framing is wrong, and that **wide structured events** (one high-cardinality event per request, queryable after the fact) subsume all three. In practice most teams still run a split stack: metrics for alerting + dashboards, traces for drill-down, logs for forensics. Know the debate; in an interview, the pragmatic answer is *"three signals with a clear drill-down path, plus exemplars to jump between them."*

> **Exemplars** — the OpenMetrics spec defines a way for a metric sample to carry a reference to a trace ID. Prometheus added experimental exemplar storage in **2.26** (behind `--enable-feature=exemplar-storage`). Clicking a latency bucket in Grafana can jump straight to a trace in that bucket. This is what makes metrics-first observability actually workable.

---

[Next: Structured Logging →](02-structured-logging.md) | [Back to index](README.md)
