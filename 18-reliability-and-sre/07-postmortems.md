# Postmortems

A postmortem is the written record of an incident and what we will change because of it. The Google SRE workbook is explicit: *"a truly blameless postmortem culture results in more reliable systems."* The purpose is learning, not punishment — and the format is deliberate. A weak postmortem produces three action items nobody does; a strong one prevents the next incident.

## Blameless — what it actually means

Blameless does **not** mean "no names" or "nobody is accountable." It means:

- Focus on **systemic gaps** that made the mistake possible, not on the person who made it.
- Assume the human acted reasonably given the information they had at the time.
- The question is not *"why did Alice deploy at 6 PM Friday?"* but *"why did our process allow a Friday-evening deploy without a peer review?"*

From the Google SRE workbook, describing what a good postmortem's authors did: *"The authors focused on the gaps in system design that permitted undesirable failure modes."* The contrast is with postmortems that target people.

Why this matters: if blame is on the table, engineers hide information. Hidden information means no learning, which means the same incident next quarter.

## The standard structure

Google's template (condensed) — adapt but don't skip sections:

```markdown
# Postmortem: <Incident short name> — YYYY-MM-DD

## Summary
One paragraph. What happened, user-visible impact, how long.

## Impact
- Users affected: X
- Duration: HH:MM
- Data loss: yes / no + details
- SLO burn: e.g. "consumed 40% of the monthly error budget"

## Timeline (UTC)
- 14:03  Deploy of service-X v2.14.0 begins
- 14:07  Error rate alert fires
- 14:09  On-call paged; IC declared
- ...

## Root cause
Technical explanation of the failure chain. Not "Alice deployed a bug" —
"A regression in v2.14.0's connection-pool config capped at 10 instead of 100; 
under normal load this throttled downstream calls; the bug was not caught 
because the integration test uses an in-memory stub."

## Contributing factors
- Deploy happened outside the normal review window.
- The dashboard showing pool saturation was broken.
- No automated rollback on error-rate spike.

## Trigger
The specific event that turned latent risk into outage.

## Resolution
What was actually done to restore service. "Rolled back to v2.13.4 at 14:21."

## Detection
How we noticed — alert, customer report, dashboard. Time-to-detect matters.

## Action items
| ID | Action | Owner | Priority | Due |
|----|--------|-------|----------|-----|
| AI-1 | Add integration test using real connection pool | @bob | P0 | 2025-09-01 |
| AI-2 | Automated rollback on 5xx spike | @team-sre | P1 | 2025-09-15 |

## Lessons learned
What went well, what went poorly, where we got lucky.
```

Action items without **owner + due date + priority** are not action items — they are wishes. The SRE workbook repeatedly stresses measurable end-states ("integration test covers real pool behavior") over vague intent ("improve testing").

## Root cause vs contributing factors

The **root cause** is the proximate technical fault. The **contributing factors** are everything around it that let the fault reach production or go undetected. Serious postmortems distinguish the two, because fixing only the root cause leaves the same latent holes for the next incident.

Example:

- Root cause: `null` deref in a payment webhook handler.
- Contributing factors: code path not covered by tests; monitoring did not alert on webhook failures; manual deploy skipped the staging canary; runbook for webhook outages did not exist.

If you only fix the null deref, the next incident will be a different bug in the same unguarded code path.

## Why "5 Whys" often fails

The 5 Whys technique — ask *"why?"* five times until you reach the "real" cause — is popular and frequently misused. Failure modes:

- **Linear thinking in a non-linear system.** Real incidents have multiple simultaneous causes; 5 Whys forces a single chain.
- **Stops at the most human-shaped answer.** *"Why did the deploy fail?" → "Because Alice pushed it."* — and analysis halts there.
- **Confirmation bias.** The first plausible chain is accepted because five levels of "why" sounds rigorous.

Better tools: a **causal diagram** (multiple parallel chains leading to the outcome) or an **STAMP / CAST**-style analysis (treats the system as a control loop and asks which controls were missing or inadequate). At minimum, separate "root cause" and "contributing factors" sections force breadth.

## Publication and follow-through

The SRE workbook stresses timeliness: *"A prompt postmortem tends to be more accurate because information is fresh in the contributors' minds."* Its good-postmortem example notes the doc was *"written and circulated less than a week after the incident was closed."* A postmortem delivered a month later is archaeology.

Two discipline points seniors notice:

1. **Action-item follow-through is reviewed.** Track AI completion rate across incidents. Unclosed AIs from three incidents ago are a signal the org does not learn from outages.
2. **Postmortems are shared broadly.** Google's practice is cross-org publication so other teams learn from incidents they did not own. Silent postmortems waste 90% of their value.

## Anti-patterns

- **"Human error" as the root cause.** Humans make errors; the question is why the system let that error reach production.
- **Action items that can't be verified.** *"Improve testing"* — how would you know when it's done?
- **Postmortems with no owner.** The IC typically owns authorship; the service owner signs off.
- **One template for everything.** A 5-minute outage and a 4-hour outage need different depth. Don't fetishize the format.

> The test of a postmortem is simple: **would a new engineer, reading it a year from now, understand what broke, why, and what we changed?** If not, rewrite it.

---

[← Previous: Incident Response](06-incident-response.md) | [Next: Error Budgets →](08-error-budgets.md) | [Back to index](README.md)
