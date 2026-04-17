# 14 - Observability

Observability is the ability to answer arbitrary questions about a running system **from its outputs alone** — without shipping new code. For a senior, "we have logs" is not an observability strategy. You are expected to reason about the three pillars (logs, metrics, traces) together, design SLIs/SLOs that reflect user experience, and build alerts that wake the on-call only for real customer pain. This section covers the .NET-native APIs (`ILogger`, `System.Diagnostics.Metrics`, `Activity`), the cross-vendor standards (OpenTelemetry, W3C Trace Context), and the operational practices (SLI/SLO/SLA, burn-rate alerting, dashboard design) that senior engineers are expected to own.

## Contents

1. [The Three Pillars](01-three-pillars.md) - Logs, metrics, traces — what each is good for, cardinality trade-offs, when to reach for which
2. [Structured Logging](02-structured-logging.md) - `Microsoft.Extensions.Logging`, Serilog, JSON logs, log levels, scopes, sampling, PII redaction
3. [Metrics](03-metrics.md) - `System.Diagnostics.Metrics` (Meter, Counter, Histogram, ObservableGauge), Prometheus exposition, RED and USE methods
4. [Distributed Tracing](04-distributed-tracing.md) - OpenTelemetry concepts (span, trace, context propagation), OTel .NET SDK, exporters (OTLP, Jaeger, Zipkin)
5. [Correlation IDs](05-correlation-ids.md) - `Activity` API in .NET, W3C `traceparent` / `tracestate`, propagation through async / messaging / HTTP, baggage
6. [SLI, SLO, SLA](06-sli-slo-sla.md) - Definitions, error budgets, burn-rate math, examples (latency SLI, availability SLI), Google SRE model
7. [Alerting and Dashboards](07-alerting-and-dashboards.md) - Symptom-based vs cause-based alerts, multi-window multi-burn-rate, dashboard design principles

---

[Back to index](../README.md)
