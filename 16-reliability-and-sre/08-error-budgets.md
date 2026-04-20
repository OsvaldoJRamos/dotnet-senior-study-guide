# Error Budgets

An error budget is the **amount of unreliability the business has accepted** — expressed as `1 - SLO`. If your SLO says *"99.9% of requests succeed over 28 days"*, the error budget is the remaining 0.1%. That number is a currency: every outage, every bad deploy, every failed experiment spends from it. Google's SRE book frames this as the mechanism that reconciles the product team's desire to ship fast with the SRE team's desire to run reliably — both are working against the same budget.

## Formal definition

From the *Embracing Risk* chapter of the Google SRE book:

> *"The difference between these two numbers [the SLO and actual availability] is the 'budget' of how much 'unreliability' is remaining for the quarter."*

And the example the book uses:

> *"imagine that a service's SLO is to successfully serve 99.999% of all queries per quarter ... the service's error budget is a failure rate of 0.001% for a given quarter."*

So the calculation is:

```text
Error budget = (1 - SLO) * time_window
```

At 99.9% availability over 28 days, the budget is ~40 minutes of downtime-equivalent per window. At 99.99%, ~4 minutes. At 99.999%, 26 seconds.

## Relationship with SLIs, SLOs, SLAs

| Term | Definition | Example |
|---|---|---|
| **SLI** — Service Level Indicator | A metric that quantifies a user-facing property | Ratio of successful HTTP 2xx/3xx responses |
| **SLO** — Service Level Objective | A target for an SLI over a window | 99.9% success over 28 rolling days |
| **SLA** — Service Level Agreement | A contractual commitment to customers, usually with penalties | 99.5% with 10% credit on breach |
| **Error budget** | `1 - SLO` over the window; the permitted unreliability | 0.1% of 28 days ≈ 40 min |

SLOs are **stricter** than SLAs. If you contract for 99.5% (SLA), run internally to 99.9% (SLO). When your SLO starts burning, you have time to react before the SLA breaks.

## Burn rate

Burn rate is **how fast you are consuming the budget relative to sustainable**. Burn rate of `1.0` means you'll use exactly the full budget by the end of the window; `10.0` means you'll exhaust it in 1/10th the window.

```text
burn_rate = (actual_error_rate) / (1 - SLO)
```

Example: SLO = 99.9% → max error rate 0.1%. If you're currently seeing 1% errors, burn rate = 10 — you'll exhaust a 28-day budget in 2.8 days. This is an alert-worthy condition *before* you actually exhaust the budget.

Multi-window multi-burn-rate alerts (Google SRE's recommended pattern) look like:

| Alert | Window (short / long) | Burn rate threshold | Page? |
|---|---|---|---|
| Fast burn | 1 h / 5 min | 14.4× | Page — will consume 2% of monthly budget in 1 h |
| Slow burn | 6 h / 30 min | 6× | Ticket — long-term issue |

The two-window approach prevents alerting on a single one-minute spike while still catching a fast, real burn.

## What triggers a feature freeze

The error-budget policy codifies what happens when the budget is exhausted. From the Google SRE book *Embracing Risk* chapter, when budgets deplete: *"releases are temporarily halted while additional resources are invested in system testing and development."*

A concrete policy looks like:

```markdown
# Error budget policy — service: checkout

- SLO: 99.9% request success, 28-day rolling window.

## Green (budget > 50%)
- Normal operations. Feature launches allowed.
- Chaos experiments encouraged.

## Yellow (25–50% budget remaining)
- Deploys continue. No large-surface migrations.
- Reliability work gets prioritized alongside features in next sprint.

## Red (<25% budget remaining)
- Freeze non-critical feature launches.
- All sprint capacity goes to reliability until green.
- Incident retros reviewed for common themes.

## Exhausted (<0)
- No feature deploys. Security patches and reliability-only.
- Post-freeze review with leadership to unblock.
```

The discipline point: the policy is written **before** an incident, signed by product + engineering leadership. Arguing policy mid-incident is how freezes get overridden and the budget becomes decoration.

## Calibration

Set the SLO where it will be **just-barely-met by a well-run service**. Two anti-patterns:

- **SLO too strict** (99.999% on a feature that can't justify the cost): always in red, team ignores the policy, budget becomes meaningless.
- **SLO too loose** (99.5% for a checkout path): budget never burns, reliability backslides, customers leave before the SLO notices.

A good heuristic: measure current p99 performance for a month, pick an SLO that is ~10–20% tighter than current, and review it every quarter. SLOs are not fixed; they evolve with the service's importance.

## What error budgets buy you

1. **Velocity vs reliability is no longer a shouting match.** Both sides work against the same number.
2. **Chaos engineering has a ceiling.** You experiment *within* the budget; when it burns, you stop.
3. **Feature teams care about reliability.** Because their shipping velocity depends on it.
4. **Outages stop being surprises.** A burn-rate alert tells you where the next freeze is coming before it lands.

## When error budgets are the wrong tool

- **Early-stage products** with no stable baseline to measure against. You don't know what your SLI should be yet — measure first, commit later.
- **Services where downtime is catastrophic** (life safety, single-shot trading). The error budget abstraction implies you accept failures at the SLO rate; some domains do not.
- **Tiny user base with spiky traffic.** The statistical math breaks down under very low request volumes; use time-based SLOs (*"uptime measured in minutes of availability"*) instead of success-ratio SLOs.

> Budgets only work if the org respects them. A freeze that leadership overrides once is a freeze that will be overridden every time. Either the policy holds, or the SLO is wrong and should be loosened — pick one, in writing.

---

[← Previous: Postmortems](07-postmortems.md) | [Back to index](README.md)
