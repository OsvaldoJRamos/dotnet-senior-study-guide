# Requirements Management

Eliciting requirements is half the job. **Managing** them over the lifetime of the project — keeping them findable, traceable, prioritized, and up to date as the business changes — is the other half. Without management, a project loses control even with good initial requirements.

## The four pillars

| Pillar | What it means | Why it matters |
|---|---|---|
| **Traceability** | Every requirement links to its source (business goal, stakeholder) and to every downstream artifact (story, code, test, deploy). | Answers *"why does this behavior exist?"* and *"what breaks if I remove it?"* |
| **Baselines & versioning** | A snapshot of the agreed set at a point in time, and a change log from there. | Gives you a reference point for change-control and audits. |
| **Change control** | A defined process to propose, assess, and approve changes. | Prevents scope creep; makes impact visible before commitment. |
| **Prioritization** | A shared method to rank requirements. | Forces trade-off discussions up front instead of during a crunch. |

## Traceability in practice

Traceability is a chain:

```
Business goal → Epic → Story → Acceptance criteria → Code → Test → Deploy
```

At each arrow, the artifact carries a link (or ID) back to the previous one. Tools that implement this natively:

- **Azure DevOps / Jira / Linear** — stories link to epics, PRs link to stories, test runs link to builds.
- **Git commit messages** that reference the story ID (`[ORD-1234] Add …`).
- **ADRs** that reference the requirements they satisfy (see [Design Docs, C4 and ADRs](../06-architecture-and-patterns/17-design-docs-c4-adr.md)).

### Why you care

- **Impact analysis.** "We need to remove the promo-code feature." Traceability tells you every story, test, and code path.
- **Audit.** Regulators ask *"show me how you verified this control"*. The trail is the evidence.
- **Institutional memory.** Years later, *"why does this weird behavior exist?"* is answered by the linked ADR, not by archaeology.

## Baselines and versioning

A **baseline** is the set of requirements agreed at a specific moment — end of discovery, start of development, pre-release. Each subsequent change is tracked against it.

Modern teams don't print-and-sign a spec; the equivalent is:

- A Git tag or release branch (`baseline/2026-Q1`).
- A Jira/ADO release snapshot.
- An ADR written at the moment of commitment.

Without a baseline, *"we agreed to this"* becomes impossible to verify after two months of refinement.

## Change control

Every non-trivial project needs a defined way to accept or reject changes:

1. **Proposed change** (written, not verbal).
2. **Impact assessment** — scope, cost, schedule, risk, which NFRs are affected.
3. **Decision** by the appropriate authority (product, steering committee, CAB for regulated systems).
4. **Recorded** as a baseline revision or an ADR.

You don't need a formal CAB for a startup feature. You do need **someone** authorized to say yes/no, and a record of what was decided. The cost of saying *"sure"* to every inbound change without triage is scope collapse mid-sprint.

## Prioritization frameworks

No team has the bandwidth for everything. The frameworks below are tools for making trade-offs **explicit**.

### MoSCoW

DSDM's categorization (`agilebusiness.org`):

| Category | Official meaning |
|---|---|
| **Must have** | *"Minimum Usable SubseT (MUST) of requirements which the project guarantees to deliver"* — no Must = no launch. |
| **Should have** | *"Important but not vital"* — the solution is still viable without them. |
| **Could have** | *"Wanted or desirable but less important"* — dropped first under pressure. |
| **Won't have this time** | *"Requirements which the project team has agreed will not be delivered (as part of this timeframe)"* — **"this time"**, not "ever". |

> Explicitly recording *"Won't have this time"* matters as much as the *Musts* — it manages expectations and keeps the scope from reopening silently.

Common misuse: classifying 90% of items as "Must". If everything is must-have, nothing is. A healthy ratio is roughly 60% Must / 25% Should / 15% Could per release.

### RICE (Intercom)

Each item gets a score:

```
RICE = (Reach × Impact × Confidence) / Effort
```

- **Reach**: how many users / transactions affected per time period.
- **Impact**: how much per user (often 0.25 / 0.5 / 1 / 2 / 3 scale).
- **Confidence**: 0.5 (low), 0.8 (medium), 1.0 (high).
- **Effort**: person-months (or person-weeks for smaller items).

Strength: puts **confidence** front and center — a high-impact item with low confidence ranks lower than a measured, high-confidence win.

Weakness: quantifies things that often aren't quantifiable. Treat the number as a discussion prompt, not a decree.

### WSJF (SAFe)

Weighted Shortest Job First — from Don Reinertsen / SAFe:

```
WSJF = Cost of Delay / Job Size
Cost of Delay = User-Business Value + Time Criticality + Risk-Reduction/Opportunity-Enablement
```

Strength: explicitly recognizes that **delay has a cost** — a feature shipped in 2 weeks is worth more than the same feature in 8.

Weakness: requires relative scoring across many dimensions — easy to pseudo-quantify opinions.

### When to use which

| Situation | Fit |
|---|---|
| Single release / scope under pressure | **MoSCoW** |
| Continuous product backlog across many features | **RICE** |
| Portfolio of initiatives, lean / SAFe shop | **WSJF** |
| Simple small team | Written rank-ordered list — no framework needed |

The best framework is the one the team actually uses. Switching tools is cheaper than letting priorities be invisible.

## Metrics that matter

A few diagnostic metrics that tell you whether requirements management is working:

| Metric | Healthy signal |
|---|---|
| **Requirement volatility** | Stable release-over-release; spikes mean unclear goals |
| **Change requests approved vs proposed** | Most proposals get a clear yes/no within a week |
| **Defects per requirement** | Low and declining; spikes expose under-specified areas |
| **Test coverage per requirement** | Every Must has at least one automated verification |
| **Requirements without owner** | Zero — every requirement has a responsible stakeholder |
| **Time-to-clarify** | Hours, not weeks — if clarifications stall, elicitation broke down |

Metrics are leading indicators. If any trends the wrong way for two sprints, dig in.

## Pitfalls

- **Verbal-only decisions.** *"We agreed in a Zoom call."* In two months, nobody remembers the exact agreement.
- **Priority by loudness.** The most insistent stakeholder gets their features first — and the most strategic ones starve.
- **Scope creep without change control.** A Slack message that turns into 2 weeks of work, unplanned.
- **Traceability as theater.** Auto-linking stories to tests that don't exercise the behavior is worse than nothing — it gives false confidence.
- **Requirements frozen forever.** The world changes. A requirement that made sense last quarter may not next quarter. Re-evaluate at baselines.
- **No owner.** A requirement nobody champions will get dropped silently; a requirement championed by one person who leaves will orphan the whole feature.

## Rule of thumb

> Requirements management isn't bureaucracy; it's **memory**. A project without baselines, traceability, and change control works until someone asks *"why did we build this?"* — and by then the answer is gone.

---

[← Previous: User Stories and BDD](04-user-stories-and-bdd.md) | [Back to section](README.md) | [Next: Communication and Audience →](06-communication-and-audience.md)
