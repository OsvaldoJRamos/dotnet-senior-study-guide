# Software Requirements Fundamentals

A **requirement** is a verifiable statement of something the system must do or be. It is the unit of agreement between everyone who pays for the system, uses it, or builds it. If a requirement cannot be verified, it isn't one — it's a wish.

The international standard for requirements engineering is **ISO/IEC/IEEE 29148** (superseding the older IEEE 830). You don't need to memorize it, but the categories below match it.

## What "good" looks like — per-requirement

A sound requirement is:

| Attribute | Meaning |
|---|---|
| **Necessary** | Removing it leaves a gap the system is supposed to fill. |
| **Unambiguous** | Exactly one reading. "Fast" is ambiguous; "p95 ≤ 300 ms at 500 rps" is not. |
| **Complete** | Doesn't rely on "and whatever else" to be interpreted. |
| **Consistent** | Doesn't contradict another requirement. |
| **Verifiable** | Can be checked by test, inspection, or measurement. |
| **Traceable** | Has an ID and links to its source (business goal, stakeholder, ADR). |
| **Feasible** | Achievable within known constraints (budget, tech, time). |

The senior-level skill isn't writing them — it's catching when one of these is missing in a backlog item before the team commits to it.

## The three classes

| Class | Answers | Example |
|---|---|---|
| **Business** | *Why* are we doing this? Strategic outcome. | "Reduce checkout abandonment by 10% this quarter." |
| **Functional (FR)** | *What* must the system do? | "The system accepts a promo code at checkout and applies the discount." |
| **Non-Functional (NFR)** | *How* must the system behave? | "Checkout p95 latency ≤ 300 ms at 1 000 rps." |

A business requirement decomposes into FRs **and** the NFRs that constrain them. All three must be aligned.

For how NFRs drive architectural decisions, see [NFR-Driven Architecture](../06-architecture-and-patterns/18-nfr-driven-architecture.md).

## Ambiguous, conflicting, implicit — the real world

Real backlogs are rarely clean. Part of the senior job is recognizing the three failure modes:

### Ambiguous

*"The report should load fast."* — Fast how? From where? For how many rows? Under what concurrency? An ambiguous requirement always costs more than the follow-up conversation does.

Fix: force a number, a percentile, and a condition. *"The report p95 loads in ≤ 2 s for up to 10 000 rows on a 4G network, at 20 concurrent users."*

### Conflicting

*"All actions must be audited to a tamper-proof log"* meets *"The homepage p95 ≤ 100 ms"*. Both are valid; together, they force a design trade-off (async audit write, CDC stream, queue in front of the audit sink).

Fix: surface the conflict **to stakeholders**, not inside the team. Pick the binding one; compensate the other with design choices.

### Implicit

Nobody writes it down but everyone expects it: *"of course we don't lose data"*, *"obviously the app has to work on mobile Safari"*, *"naturally it's LGPD-compliant"*. These are the ones that cause production incidents.

Fix: maintain a checklist of "default" NFRs (availability, backup/restore, security, observability, accessibility, compliance) and confirm them on every epic. If you don't elicit them, the absence shows up in production.

## The lifecycle

Requirements are not static. They go through predictable stages:

```
Elicited → Analyzed → Specified → Validated → Approved → Implemented → Verified → Maintained
```

At each transition something should change — the artifact gains an ID, gets linked to tests, gets a version. The lifecycle is what prevents "we all agreed to this in a meeting in March" becoming lost context in July.

See [Requirements Management](05-requirements-management.md) for the mechanics of traceability, baselines, and change control.

## Cost of a bad requirement

The earlier a defect is caught, the cheaper it is — a principle going back to Barry Boehm's *Software Engineering Economics* (1981) and reproduced in many secondary studies since. The exact multipliers vary widely between sources (some cite 5×, others 10× or 100× when a defect escapes to production) and depend on the process being measured — but the **shape of the curve is robust**: defects caught at requirements time are cheap; defects caught after release are orders of magnitude more expensive.

This is why *"we'll figure out the requirement later"* is the most expensive sentence in a project. A missing NFR that you discover the week before launch reshapes the architecture. A missing FR that you discover after launch costs rework, hotfixes, and user trust.

## Senior role

At a senior level you're not writing every requirement — you're:

1. **Spotting missing requirements** (especially implicit NFRs) before commitment.
2. **Pushing back on ambiguity** with concrete numbers.
3. **Mediating conflicts** between business, compliance, UX, and engineering.
4. **Linking requirements to architecture** — each binding NFR becomes an ADR.
5. **Protecting the team from scope creep** by insisting on change control.

## Pitfalls

- **Confusing solution with requirement.** *"Use Kafka to handle order events"* is a solution. The requirement is *"Order events must be durable, replayable, and consumable by ≥ 4 services with back-pressure."* Solutions belong in design; requirements describe needs.
- **Accepting "just like the old system".** The old system encoded years of accidents. Re-elicit. Don't port cruft.
- **Signing off on a backlog item with no acceptance criteria.** You'll write them in your head; the next engineer will write them differently; QA will write a third version.
- **Treating NFRs as "nice to have".** Availability and latency are the constraints the architecture is *built* around; they aren't optional.
- **Freezing requirements on day one.** They will change. The question is whether the change goes through a review or through a Slack message in a private DM.

## Rule of thumb

> A good requirement survives the question *"how will we test this?"*. If the answer is vague, the requirement is not finished.

---

[← Back to section](README.md) | [Next: Requirements Elicitation →](02-requirements-elicitation.md)
