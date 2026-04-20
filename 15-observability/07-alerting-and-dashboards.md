# Alerting and Dashboards

The test of an observability stack is not whether it collects data, but whether it **wakes the right person at the right time** and shows the right thing on the screen when they arrive. Alerting and dashboards are where observability meets the on-call human.

## Symptom-based vs cause-based alerts

**Symptom-based alerts** fire on things a user feels: latency, error rate, job freshness, dropped messages.

**Cause-based alerts** fire on things that *might* cause user pain: CPU high, disk low, thread pool starvation, queue depth.

The senior rule (from Google SRE's *Monitoring Distributed Systems* and *Alerting on SLOs*): **page on symptoms, not on causes.** Cause alerts are noisy — the cause can exist without customer impact (auto-scaling handled it; traffic was low; the circuit breaker stopped cascading). If nobody is being hurt, you shouldn't wake someone up.

Causes still matter — they belong in **dashboards** and **tickets**, not pages. A graph of thread pool queue depth is perfect for the investigation; an alert on thread pool queue depth at 03:00 usually isn't.

## SLO-based alerting — the modern default

The framing from Google's SRE Workbook, *Alerting on SLOs*, is: **alert on burn rate of your error budget, not on raw error rate.**

Why:
- **Severity tracks impact.** Burning 2 % of the monthly budget in an hour is a different problem from burning 2 % in two weeks.
- **Reset is fast.** An alert that fires on "5× error rate right now" can flap. An alert on "14.4× burn rate sustained over 1 h AND over 5 min" stops paging within 5 minutes of recovery.
- **The math composes.** One rule parametrized by burn-rate threshold and windows serves both pages and tickets.

### The recommended shape — multi-window, multi-burn-rate

From the SRE Workbook (Table 5-8) for a 99.9 % SLO:

```
Page       IF  (burn_rate_over_1h  > 14.4)  AND  (burn_rate_over_5m  > 14.4)
Page       IF  (burn_rate_over_6h  >  6)    AND  (burn_rate_over_30m >  6)
Ticket     IF  (burn_rate_over_3d  >  1)    AND  (burn_rate_over_6h  >  1)
```

Both windows must exceed the threshold for the alert to fire. The long window filters noise; the short window ensures recovery is detected quickly.

### Implementing it in Prometheus

```yaml
# Error-rate fraction — your SLI denominator
- record: job:sli_errors:ratio_rate5m
  expr: |
    sum(rate(http_requests_total{job="orders-api",code=~"5.."}[5m]))
    /
    sum(rate(http_requests_total{job="orders-api"}[5m]))

# Burn rate = SLI error ratio / (1 - SLO target)
- alert: OrdersApiErrorBudgetBurningFast
  expr: |
    (
      job:sli_errors:ratio_rate1h > (14.4 * 0.001)
      AND
      job:sli_errors:ratio_rate5m > (14.4 * 0.001)
    )
  for: 2m
  labels:
    severity: page
  annotations:
    summary: "Orders API burning error budget 14.4x over last hour"
    runbook: "https://runbooks.example.com/orders-api/error-budget-burn"
```

> Every paging alert should have a **runbook link** in its annotations. An alert without a runbook is a bug.

## Alert quality rules of thumb

From SRE literature and practice:

- **Every page is actionable.** If the response is "acknowledge and go back to sleep", the alert is wrong. Tune it or delete it.
- **Every page is novel.** Repeated identical pages → consolidate into one alert with a count, or auto-remediate.
- **Every page is urgent.** Not-urgent = ticket, not page.
- **Alert groups > single alerts.** Group by incident (service, cluster) so a DB outage produces one page, not 40.
- **Low signal-to-noise** destroys on-call. Target a page frequency you'd be willing to be on-call for.

## The Four Golden Signals — the baseline any service should alert on

From the SRE book's *Monitoring Distributed Systems* chapter:

| Signal | Definition |
|---|---|
| **Latency** | "The time it takes to service a request." Measure successful and failed separately — a fast 500 is still a failure. |
| **Traffic** | Demand on the system (RPS, tx/s, I/O rate). |
| **Errors** | "The rate of requests that fail, either explicitly (HTTP 500s), implicitly (HTTP 200 with wrong content), or by policy." |
| **Saturation** | How full the service is. "Many systems degrade in performance before they achieve 100 % utilization." |

If you do nothing else, alert on latency + errors (symptom) and dashboard traffic + saturation (cause).

## Dashboard design principles

A dashboard is not a database browser. It is a **decision-support tool** for a specific audience answering a specific question.

### The three-tier model

1. **Service overview** — one screen, SLO status + Four Golden Signals for this service. The on-call sees this first.
2. **Subsystem drill-down** — per-component health (DB pool, cache, queue). You land here after the overview tells you something is off.
3. **Raw exploration** — ad-hoc queries in the metric explorer, trace viewer, log search. Where you end up after the subsystem dashboard shows the anomaly.

Each tier should **link to the next**. Grafana "data links" or Azure Monitor workbooks do this natively.

### Design rules

- **One headline per dashboard.** The top row answers the single question the dashboard exists for: *"are we meeting SLO?"* Everything else is supporting.
- **Time range consistent across panels.** Use a single shared variable, not per-panel overrides.
- **Axes start at zero** for counters/rates unless you have a reason. An error rate chart that doesn't start at zero exaggerates small fluctuations.
- **Latency is a distribution, not a line.** Show at least p50, p95, p99 together — and ideally a full heatmap. A p95 line alone hides multimodal behavior.
- **Show the target.** If the SLO is "p95 < 300 ms", draw a line at 300 ms. If a metric has a meaningful threshold, draw it.
- **Annotate deploys.** Every production dashboard should show deploy markers. *"Did p99 jump at 14:03? What deployed at 14:02?"* is the most common incident question.
- **Lead with RED, follow with USE.** User-facing services want the RED method (Rate / Errors / Duration) at the top. Resources (DB, cache, disk) want USE (Utilization / Saturation / Errors).

### Anti-patterns

| Anti-pattern | Why it breaks |
|---|---|
| *"Overview"* dashboard with 40 panels | Nobody reads past row 2 at 03:00. Split into focused dashboards. |
| All metrics on one time range, no comparison | You can't tell if "500 errors/min" is normal or a spike. Add a "7 days ago" reference line or use `rate - offset 7d`. |
| **Averages for latency** | Hides tail behavior. A service with 99 % requests at 20 ms and 1 % at 30 s has a totally acceptable-looking average. |
| No units, no thresholds | A green number in a blue box is not observability. |
| Count instead of rate | `http_requests_total` is not meaningful; `rate(...)` is. |

## Platform notes

| Platform | Dashboard tool | Alerting |
|---|---|---|
| **Grafana / Prometheus** | Grafana | Prometheus Alertmanager (route, group, silence) or Grafana Alerting. |
| **Azure** | Azure Monitor Workbooks, Grafana on Azure | Azure Monitor alert rules (metric, log, activity). |
| **AWS** | CloudWatch Dashboards, Grafana | CloudWatch Alarms, SNS → PagerDuty/Opsgenie. |
| **Datadog / New Relic** | Native dashboards | Native monitors; good OOTB golden-signal monitors. |

The portability story is the same as for exporters: instrument with OTel, ship OTLP to a Collector, let the Collector route to the vendor. That way your instrumentation code is vendor-neutral; your dashboards and alerts are the vendor-locked layer (that's unavoidable).

## What to alert on — a starter set for a typical .NET web service

Every user-facing endpoint:
1. **SLO burn rate** — multi-window multi-burn-rate on the availability SLO (page).
2. **SLO burn rate** — on the latency SLO (page or ticket, depending on criticality).

Supporting (ticket, not page):
3. **DB connection pool saturation** — `pool_in_use / pool_max > 0.9` for 10 min.
4. **Dependency circuit breaker open** — any breaker open for > N minutes.
5. **Dead-letter queue depth growing** — not decreasing for N minutes.
6. **Cert expiry** — any cert < 14 days from expiring.
7. **Deploy anomaly** — error rate post-deploy significantly higher than pre-deploy baseline.

Not pages (dashboard only):
- CPU, memory, pod restarts, GC time, thread pool starvation. These are *diagnostic* unless they cause an SLI breach.

## What NOT to do

- Don't alert on `ERROR` log patterns. Convert to a counter, alert on the counter's rate.
- Don't have per-host / per-pod alerts. Alert on the service aggregate; let orchestration handle individual instances.
- Don't set the page threshold to "two consecutive errors". Use burn rate.
- Don't add an alert without a runbook. An alert that doesn't tell you what to do at 03:00 is not an alert; it's a notification.
- Don't build dashboards before you know who reads them. A dashboard nobody opens is dead weight; trim quarterly.

---

[← Previous: SLI, SLO, SLA](06-sli-slo-sla.md) | [Back to index](README.md)
