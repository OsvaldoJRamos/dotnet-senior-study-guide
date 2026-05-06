# Incident Response

An incident is an unplanned event that degrades a user-visible service beyond acceptable limits. The point of a formal response process is not paperwork — it is **to stop the freelancing**. Without roles, engineers make independent changes that conflict, conversations scatter across five channels, and nobody is tracking what has been tried. Google SRE's chapter on managing incidents describes exactly this failure mode as the motivation for the Incident Command System.

## The Incident Command System (IC)

Adapted by Google SRE from the fire-service ICS. Four roles; one person can hold more than one only in small incidents.

| Role | Responsibility |
|---|---|
| **Incident Commander (IC)** | Holds the high-level state; delegates; runs the bridge. Does **not** execute fixes. |
| **Ops Lead** | The only person / team that changes the system during the incident. Everyone else stays hands-off. |
| **Communications Lead** | Talks to users, stakeholders, the status page; keeps the incident document current. |
| **Planning Lead** | Files bugs, tracks resources, handles long-horizon concerns (handoffs, follow-ups). |

From the Google SRE book, *Managing Incidents* chapter: *"The operations team should be the only group modifying the system during an incident."* This is the rule that prevents two engineers from simultaneously rolling back and pushing a hotfix and making everything worse.

## Severity levels

Most orgs use four tiers. Exact thresholds vary, but the shape is consistent:

| Severity | Example | Response |
|---|---|---|
| **SEV-1** | Full outage; data loss; security breach in progress | Page on-call + IC; war room now; status page red; hourly comms |
| **SEV-2** | Major feature broken for most users; SLO burn critical | Page on-call; IC assigned; status page yellow; comms every 2 h |
| **SEV-3** | Minor feature degraded; clear workaround exists | Normal working hours; ticketed; status page optional |
| **SEV-4** | Cosmetic; internal-only; no user impact | Backlog item |

Two common mistakes: calling everything SEV-1 (the team burns out, the signal becomes noise), and waiting for certainty before declaring (declare early, downgrade if it's not that bad).

## Runbooks

A **runbook** is a scenario-specific operational playbook. Not generic advice — specific steps for a known failure mode, linked from the alert that would fire.

Good runbook:

```markdown
# Runbook: Payments service p99 > 1s

## Detect
Alert: `payments_p99_above_threshold` (fires after 5 minutes)

## Immediate checks
1. Dashboard: [payments-overview]
2. Is the circuit breaker open? Check metric `payments_cb_state == open`
3. Recent deploys? `kubectl rollout history deploy/payments`

## Mitigations (in order of preference)
1. Rollback if deploy happened in last 60 min: `kubectl rollout undo deploy/payments`
2. Scale out: `kubectl scale deploy/payments --replicas=20`
3. Failover to us-west-2: set `active_region=us-west-2` in LaunchDarkly

## Escalation
If not mitigated in 15 min → page `payments-oncall-senior`
```

Bad runbook: *"investigate the issue, check logs, fix it."* — that is the job description, not a runbook.

## Communication cadence

Two audiences, two cadences:

- **Internal war room** — continuous, rolling updates in the incident channel. Every action typed (`"rolling back to v1.2.3"`), every observation (`"error rate dropped from 12% to 3%"`).
- **External + leadership** — scheduled updates regardless of whether there is news. *"Still investigating; no new mitigations attempted in the last 30 minutes; next update at 14:30 UTC."* Silence is worse than "no change" — silence makes users assume the worst.

## Status pages

A customer-facing status page is an honest contract: if your own page says green during a SEV-1, you have damaged trust *twice* — once for the outage, once for lying about it.

Good practice:

- One page per logical component (API, Dashboard, Webhooks — not just "everything").
- Update it **during** the incident, not after.
- Subscribe to updates for the services you depend on (AWS, Stripe, Auth0 ...); mirror their status in yours when relevant.

## Handoffs across time zones

Google SRE calls out handoffs explicitly: long incidents cross shifts and the IC baton must transfer cleanly. A handoff is a short meeting, not a Slack message:

1. State of the incident (1 min).
2. What's been tried; what worked; what made it worse.
3. Open threads and who is waiting on what.
4. Next planned action.
5. Formal "you have IC" from outgoing, "I have IC" from incoming.

The **living incident document** is the artifact that makes this work — a single shared doc the IC keeps current, linked from the channel topic.

## Common anti-patterns

- **Everyone is IC** — nobody is.
- **The IC is also fixing things.** You can hold the state or fix the bug, not both.
- **Parallel channels.** One channel per incident. Cross-posting to Slack + email + two Teams rooms means nobody has the full picture.
- **Heroics over process.** The goal is restoring service fast *and* learning. Waking the one person who knows the system once is fine; organizing around that person is a bus factor of one.

---

[← Previous: Chaos Engineering](05-chaos-engineering.md) | [Next: Postmortems →](07-postmortems.md) | [Back to index](README.md)
