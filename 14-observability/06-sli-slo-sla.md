# SLI, SLO, SLA

Three words that get confused in every interview. The Google SRE book (*Service Level Objectives* chapter, Chris Jones et al.) gives the clean definitions and they are worth memorizing verbatim.

## The definitions, from the source

> **SLI** — *"a service level indicator: a carefully defined quantitative measure of some aspect of the level of service that is provided."*
>
> **SLO** — *"a service level objective: a target value or range of values for a service level that is measured by an SLI."* Structure: `SLI ≤ target`, or `lower ≤ SLI ≤ upper`.
>
> **SLA** — *"service level agreements: an explicit or implicit contract with your users that includes consequences of meeting (or missing) the SLOs they contain."*

The question that disambiguates SLO vs SLA: *"what happens if we miss it?"* No consequence → SLO. Refund, credit, contract penalty → SLA.

## What's actually in an SLI

An SLI is a **ratio of good events to valid events**, computed over a window. Not a raw metric.

| Aspect | SLI example |
|---|---|
| **Availability** | `successful_requests / valid_requests` over 30 days. "Valid" excludes bad client requests (4xx you meant to reject). |
| **Latency** | `requests faster than 300 ms / valid_requests`. Note: **not** the average — SLO latency is always a ratio. |
| **Correctness** | `correct_responses / valid_requests`. Requires knowing ground truth; often shadow-traffic tested. |
| **Freshness** | `records_updated_within_5min / records`. For data pipelines. |
| **Coverage** | `successfully_processed_records / total_records`. For batch jobs. |

The SRE book is explicit: **don't average, don't p99 alone** — SLIs should be expressed as a fraction of "good" events, because that's what composes with error budgets.

## Error budgets — why SLOs are not 100 %

> *"Rather than requiring 100% SLO compliance, the document introduces the concept of allowing an error budget — a rate at which the SLOs can be missed — and track that on a daily or weekly basis."*

If your SLO is **99.9 %**, your error budget is **0.1 %** of valid events per window. For a 30-day window that's about **43.2 minutes of total budget** for availability.

Error budgets make a blameless, quantitative argument you can use with product:

- Budget fully spent → **freeze risky changes**. Ship reliability work instead of features.
- Budget healthy → **spend it**. Ship the risky experiment, do the migration, take the downtime window.

This aligns dev and SRE. 100 % reliability is the wrong target — it leaves no room to change anything.

## Picking the SLI — the right side of "fast/slow" and "up/down"

Choose at the **user-perceived boundary**, not at the internal service boundary. A senior mistake is setting an SLI on *"database latency"* — the user doesn't see that; they see the checkout page. The correct place for most user-facing systems is the **HTTP/gRPC ingress** for the critical user journey.

Examples by system:

| System | Good SLI | Bad SLI |
|---|---|---|
| Public web API | `ratio of 2xx/3xx responses to non-4xx requests, within 200 ms, on the /checkout route` | Raw CPU % on the API pod |
| Async worker | `ratio of events processed within 5 s of enqueue` | Worker "uptime" |
| Data pipeline | `ratio of records produced to records expected, per hour` | Job completion count |
| Search | `ratio of searches returning results within 300 ms and correct top-1` | QPS |

Correctness SLIs are under-used. A 200 OK with the wrong result is worse than a 500.

## Nines in practice

Widely referenced availability nines (these are industry convention, not defined in any one spec):

| SLO | Error budget per 30 days | Error budget per year |
|---|---|---|
| 99 % | 7 h 12 m | 3 d 15 h |
| 99.9 % | 43 m 12 s | 8 h 45 m |
| 99.95 % | 21 m 36 s | 4 h 22 m |
| 99.99 % | 4 m 19 s | 52 m 35 s |
| 99.999 % | 25.9 s | 5 m 15 s |

> **Interview tip:** always ask back *"what's the user-visible cost of each nine?"* 99.999 % on an internal admin tool is absurd; 99.99 % on a trading system might not be enough.

## Burn rate — how fast you're consuming the budget

**Burn rate** = (fraction of budget consumed in a window) ÷ (fraction of the SLO window that window covers).

For a 30-day 99.9 % SLO, burn rate 1× means you'd spend the entire 30-day budget in exactly 30 days. Burn rate 14.4× means you'd spend it in about 2 days.

Burn rate is the correct thing to alert on — not "instantaneous error rate", not "got 10 errors in a row". Alerting on burn rates is what the *Alerting on SLOs* chapter of the Google SRE Workbook recommends.

## Multi-window, multi-burn-rate — the recommended alert shape

The SRE Workbook (Table 5-8) recommends, for a 99.9 % SLO, these threshold pairs:

| Severity | Long window | Short window | Burn rate | % of budget at fire |
|---|---|---|---|---|
| **Page** | 1 hour | 5 minutes | 14.4× | 2 % |
| **Page** | 6 hours | 30 minutes | 6× | 5 % |
| **Ticket** | 3 days | 6 hours | 1× | 10 % |

The alert fires **only when BOTH windows exceed the threshold**. The long window makes the alert meaningful; the short window (typically 1/12 of the long) ensures the alert **resets quickly** when the incident ends — so the on-call isn't paged for something already fixed.

More on these in the next file.

## Example: two concrete SLOs

### Availability SLO — checkout API

- **SLI:** `good = (HTTP 2xx, 3xx, 4xx-that-are-client-error-not-server) on POST /checkout`, `valid = all requests on POST /checkout where auth was valid`. SLI = `good / valid`.
- **SLO:** 99.95 % over rolling 30 days.
- **Error budget:** 21 m 36 s per 30 days.
- **SLA:** 99.9 % in the customer contract — tighter internal SLO than SLA by convention, so the SLA is still met when the SLO is just missed.

### Latency SLO — search results

- **SLI:** `good = requests where response_time ≤ 300 ms`, `valid = requests on GET /search`. SLI = `good / valid`.
- **SLO:** 99 % of search requests ≤ 300 ms over rolling 28 days.
- This is why SLOs are ratios, not p99: it composes trivially with burn-rate math; p99 doesn't.

## What NOT to do

- Don't set an SLO on a metric your users don't feel ("pod restart rate").
- Don't set the SLO = SLA. You need internal headroom or you'll pay penalties on every normal-size incident.
- Don't promise 99.999 % when your dependencies don't. A service that depends on a 99.9 % cloud database cannot itself be 99.99 % on its own.
- Don't confuse an **SLO miss** with an **outage**. Missing 99.9 % by 1 minute is not a Sev-1.
- Don't publish the internal SLO as the SLA — you lose flexibility and get sued.

---

[← Previous: Correlation IDs](05-correlation-ids.md) | [Next: Alerting and Dashboards →](07-alerting-and-dashboards.md) | [Back to index](README.md)
