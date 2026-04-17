# Chaos Engineering

Chaos engineering is not "breaking things for fun" — it is **running controlled experiments that build confidence in how the system behaves under turbulent conditions**. The practice started at Netflix, was formalized in 2015 at *principlesofchaos.org*, and is now table stakes for SRE orgs. The interview-level point: chaos engineering is a scientific method applied to reliability.

## Definition (principlesofchaos.org, verbatim)

> *"Chaos Engineering is the discipline of experimenting on a system in order to build confidence in the system's capability to withstand turbulent conditions in production."*

## The five advanced principles

From principlesofchaos.org, in the order the site lists them:

| # | Principle | What it means in practice |
|---|---|---|
| 1 | **Build a Hypothesis around Steady State Behavior** | Pick a measurable system-level metric (successful checkouts / minute, p99 latency, queue depth). Predict what normal looks like before you inject anything. |
| 2 | **Vary Real-world Events** | Simulate things that actually happen: host failure, zone outage, slow disk, DNS flap, 500 responses from a dependency, clock skew. Not just "kill one pod." |
| 3 | **Run Experiments in Production** | The only environment that is actually production. Staging is a lie about production. |
| 4 | **Automate Experiments to Run Continuously** | Manual chaos runs once; automated chaos runs every day. |
| 5 | **Minimize Blast Radius** | Start with the smallest slice that can prove or disprove the hypothesis. Expand as confidence grows. |

> Principle 5 is the safety clause that makes principle 3 acceptable. *"Minimize blast radius"* means a canary cluster, a small percentage of traffic, a shadow population — never *"turn off the payments DB on Friday afternoon."*

## The experiment loop

```text
1. Define steady state         "99.95% of checkouts succeed"
2. Hypothesis                  "Losing one order-service replica does not change steady state"
3. Introduce real-world event  "Terminate one pod in the us-east-1a replica set"
4. Try to disprove             "Did the success rate drop?"
5. Record + act                "Steady state held"  OR  "Dropped by 3% for 40 s — bug in retry budget"
```

Crucially, the hypothesis is the thing you're testing. "Let's see what happens" is not chaos engineering — that is just outage-by-amateur.

## Game days

A **game day** is a scheduled, scoped, facilitated chaos exercise. Teams assemble, a facilitator injects a failure (often from a runbook), participants respond as they would in a real incident. The value is not just finding bugs — it's exercising the people, pages, dashboards, and runbooks before they're needed at 3 AM.

Senior signal at interview: *"we run quarterly game days on our top three failure modes; every finding becomes either an action item on a runbook or a ticket."*

## Tools

| Tool | What it is |
|---|---|
| **Netflix Chaos Monkey** (OSS) | The original — randomly terminates instances in an auto-scaling group. Version 2 integrates with Spinnaker; the broader Simian Army suite from Netflix is no longer actively maintained. |
| **AWS Fault Injection Service (FIS)** | Managed AWS service; formerly *Fault Injection Simulator*. Runs experiments against EC2, ECS, EKS, RDS, networking. Enforces stop conditions via CloudWatch alarms. |
| **Azure Chaos Studio** | Managed Azure equivalent; integrates with Azure resource types and Azure Monitor. |
| **Gremlin / Chaos Mesh / Litmus** | Third-party / OSS platforms with a broader fault library (latency injection, packet loss, CPU hogs, disk fill). |

## AWS FIS in one paragraph

From the official user guide: *"AWS Fault Injection Service (AWS FIS) is a managed service that enables you to perform fault injection experiments on your AWS workloads."* The core concepts are:

- **Experiment template** — the blueprint (actions, targets, stop conditions).
- **Actions** — what happens (stop an EC2 instance, inject API error, add network latency).
- **Targets** — which resources (by ARN, by tag, by filter).
- **Stop conditions** — CloudWatch alarms that abort the experiment if blast radius expands.

AWS is explicit: *"AWS FIS carries out real actions on real AWS resources in your system."* The user guide recommends a planning phase and pre-production trials before running experiments in production.

## When NOT to do chaos engineering

- **Before you have observability.** You cannot measure steady state without metrics, traces, and logs. Chaos on a blind system is just damage.
- **Before you have a working incident response.** If on-call cannot respond to a real outage today, injecting one is cruel.
- **On someone else's system** without their consent — including managed SaaS you depend on.

> Chaos engineering is the **final** maturity step. SLOs → error budgets → observability → runbooks → game days → continuous chaos. Running it out of order produces noise, not learning.

---

[← Previous: Timeouts](04-timeouts.md) | [Next: Incident Response →](06-incident-response.md) | [Back to index](README.md)
