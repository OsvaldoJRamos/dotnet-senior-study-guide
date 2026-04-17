# Stakeholders

A **stakeholder** is anyone whose interests are affected by the system — they pay for it, use it, maintain it, regulate it, or are downstream of what it does. A senior engineer who ships a technically correct system that no stakeholder wants has wasted the quarter.

## The typical cast

| Role | Cares about |
|---|---|
| **Sponsor / Executive** | Outcome, ROI, strategic alignment |
| **Product Owner / Manager** | Scope, priorities, user value |
| **End user** | Daily usability, performance, not losing their work |
| **Customer / Buyer** | Price, features delivered, support |
| **Operations / SRE** | Reliability, observability, deployability, on-call burden |
| **Security** | Threat model, auditability, least privilege |
| **Compliance / Legal** | Regulatory fit (LGPD, SOC 2, HIPAA), data residency, retention |
| **QA** | Testability, regression risk, release stability |
| **Support / Customer Success** | Diagnosability, error messages, self-service |
| **Partner / Integrator** | Contract stability, SLAs, versioning |
| **Data / Analytics** | Event completeness, schema stability, retention |
| **Other engineering teams** | Contract clarity, cross-team dependencies |

Not every project has every stakeholder. Senior engineers spot the **missing** ones — the ones nobody thought to invite until they're blocking a launch.

## Stakeholders have conflicting interests — that's normal

Designing a feature almost always runs into conflicts:

| Stakeholder A wants | Stakeholder B wants | Conflict |
|---|---|---|
| Detailed audit log of every action | p95 latency ≤ 200 ms | Synchronous logging vs performance |
| Strict role-based access | Self-service onboarding | Administrative overhead vs adoption |
| Stable API contract | Fast iteration | Old clients break on change |
| Data residency in EU | Global performance | Extra region, extra cost |
| Detailed analytics | GDPR-compliant retention | Keep forever vs delete after N days |

The job isn't to pretend the conflict doesn't exist — it's to **name it, surface it to decision-makers, and record the decision** (usually as an [ADR](../06-architecture-and-patterns/17-design-docs-c4-adr.md)).

## Mapping influence — the Mendelow matrix

Aubrey Mendelow's 1991 power-interest grid is the standard tool for stakeholder analysis. You place each stakeholder in one of four quadrants:

```
              LOW INTEREST         HIGH INTEREST
  HIGH   ┌────────────────────┬────────────────────┐
  POWER  │                    │                    │
         │     Keep           │     Manage         │
         │     Satisfied      │     Closely        │
         │                    │                    │
         ├────────────────────┼────────────────────┤
  LOW    │                    │                    │
  POWER  │     Monitor        │     Keep           │
         │                    │     Informed       │
         │                    │                    │
         └────────────────────┴────────────────────┘
```

- **Manage Closely** (high power + high interest): the CFO sponsoring a billing rewrite. Frequent, detailed communication. Involve them in decisions.
- **Keep Satisfied** (high power + low interest): the CTO who cares that the project exists but not about weekly details. Short, high-signal updates.
- **Keep Informed** (low power + high interest): end users and on-call engineers. Newsletters, changelogs, release notes.
- **Monitor** (low power + low interest): tangential teams. Occasional check-ins; re-evaluate if the project scope shifts.

A stakeholder's quadrant **changes** — a team that didn't care last month can become "manage closely" once the feature starts touching their data. Re-map whenever the project pivots.

## The Stakeholder Register

For anything beyond a small team project, keep a simple register:

| Name / Role | Concerns | Influence | Communication cadence | Last update |
|---|---|---|---|---|
| Ana — Head of Payments | Chargeback rate, reconciliation | High | Weekly 15 min | 2026-04-12 |
| João — On-call SRE | Observability, rollback story | Medium | Async (Slack), ADR reviews | 2026-04-15 |
| Legal — LGPD | PII retention, audit trail | High (blocker power) | Monthly + at every schema change | 2026-03-30 |

This isn't bureaucracy — it's how you avoid discovering on launch day that Legal never signed off on the retention policy.

## Communication styles

Different stakeholders want different things; the same information presented three different ways.

| Audience | Format | Focus |
|---|---|---|
| Executive | 1-slide, outcome-first | Did we hit the goal? Any risk to the next one? |
| Product | Brief, user-impact | What shipped, what's next, what's blocked |
| Engineering | RFC / ADR / code | Detail, trade-offs, rationale |
| Operations | Runbook / dashboard / postmortem | How to operate, what to do when it breaks |
| Legal / compliance | Controls matrix / audit doc | What data, what protection, what evidence |

See [Communication and Audience](06-communication-and-audience.md) for the craft of adjusting the message.

## Senior-level stakeholder habits

- **Over-communicate at the boundaries.** Kickoff, mid-course correction, and launch readiness are the moments where missing a stakeholder hurts most.
- **Write decisions down.** Verbal agreements evaporate; ADRs and register entries survive role changes.
- **Translate, don't filter.** Executives don't need SQL, but they do need to understand a 4-week slip. Simplify; don't hide.
- **Name the dissenter.** If a stakeholder disagrees with the direction, surface it in the decision doc ("Security raised concern X; mitigated by Y"). Silent opposition becomes loud resistance at the worst moment.
- **Consent ≠ enthusiasm.** A stakeholder saying *"sure, fine"* in a crowded meeting is not approval. Confirm in writing.

## Pitfalls

- **Stakeholder by title only.** The person in the role may not be the one with the knowledge or the blocking power. Map by actual influence, not by org chart.
- **Forgetting non-human stakeholders.** Compliance standards, SLAs to partners, API consumers — these have "interests" that must be honored even without a voice in the room.
- **Confusing PO with user.** The Product Owner **represents** the user — they are not the user. When you can, interview users directly.
- **"We'll loop them in later."** The later you loop in Security, Legal, or Ops, the more rework their feedback costs.
- **Treating the Mendelow grid as permanent.** Re-map it at every phase change.

## Rule of thumb

> Every decision an engineer makes has a stakeholder somewhere who cares about it. The job isn't to please all of them — it's to identify them, understand their concerns, and **make the trade-offs explicit** so the right person can make the call.

---

[← Previous: Requirements Elicitation](02-requirements-elicitation.md) | [Back to section](README.md) | [Next: User Stories and BDD →](04-user-stories-and-bdd.md)
