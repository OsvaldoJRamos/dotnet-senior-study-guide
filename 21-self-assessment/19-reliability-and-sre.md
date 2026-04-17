# Reliability and SRE

> Read the questions, think about your answer, then click to reveal.

---

### 1. When is a retry harmful, and what controls do you add to prevent retry storms?

<details>
<summary>Reveal answer</summary>

Retries are harmful when failures are caused by **overload** — more retries = more load = longer outage. AWS's Builders' Library quantifies the blast: a 5-layer stack each retrying 3 times can amplify backend load 243-fold.

Controls:
1. **Retry only transient errors** — never on 4xx (except 408/429), never on business rule failures.
2. **Exponential backoff with full jitter**: `sleep = random(0, min(cap, base * 2^attempt))`.
3. **Bound attempts** — 3 is a sane default; more is rarely better.
4. **Retry budget / token bucket** — every successful call refills; retries drain. When empty, retries are rejected locally. AWS SDKs have done this since 2016.
5. **Idempotency-first** — retries on a non-idempotent `POST` without an idempotency key double-charge customers.

Polly: `AddRetry` with `BackoffType = Exponential`, `UseJitter = true`, `MaxRetryAttempts = 3`. For HTTP specifically, `AddStandardResilienceHandler` gets these defaults right.

Deep dive: [Retries, Backoff and Jitter](../15-reliability-and-sre/01-retries-backoff-jitter.md)

</details>

---

### 2. Explain full jitter vs equal jitter vs decorrelated jitter and which you'd pick by default.

<details>
<summary>Reveal answer</summary>

From the AWS Architecture Blog *Exponential Backoff And Jitter* (Marc Brooker):

- **Full jitter**: `sleep = random(0, min(cap, base * 2^attempt))` — randomizes the whole delay. AWS's test found it used the **least total work**.
- **Equal jitter**: `sleep = cap/2 + random(0, cap/2)` — keeps a minimum delay. AWS found it was the **worst performer** of the three (still better than no jitter).
- **Decorrelated jitter**: `sleep = min(cap, random(base, 3 * last_sleep))` — self-correlates with recent sleep; close to full jitter in performance.

Default pick: **full jitter**. Only deviate if you have a specific reason to keep a minimum delay (e.g. protecting a peer that genuinely needs breathing room even after a fast failure).

Deep dive: [Retries, Backoff and Jitter](../15-reliability-and-sre/01-retries-backoff-jitter.md)

</details>

---

### 3. Walk me through the Polly v8 circuit-breaker state machine and its default thresholds.

<details>
<summary>Reveal answer</summary>

Four states (Polly v8 `CircuitBreakerStrategyOptions`):

- **Closed** — normal; failures counted within `SamplingDuration`.
- **Open** — short-circuits with `BrokenCircuitException` for `BreakDuration`.
- **HalfOpen** — one probe call; success closes, failure re-opens.
- **Isolated** — manually opened; throws `IsolatedCircuitException`.

Defaults (from `pollydocs.org`):

| Property | Default |
|---|---|
| `FailureRatio` | 0.1 (10%) |
| `MinimumThroughput` | 100 |
| `SamplingDuration` | 30 s |
| `BreakDuration` | 5 s |

The critical nuance: the circuit opens only when **both** conditions are met — minimum throughput reached AND failure ratio exceeded. A low-traffic endpoint with one failure won't trip, which is what you want.

Pipeline ordering: **Timeout → Retry → Circuit Breaker → call**. The breaker should see the results of retries; otherwise a single retry burst can't trip what is a real outage.

Deep dive: [Circuit Breaker](../15-reliability-and-sre/02-circuit-breaker.md)

</details>

---

### 4. Polly v8 "doesn't have a bulkhead" — how do you get bulkhead isolation today?

<details>
<summary>Reveal answer</summary>

Polly v7 had a distinct `Bulkhead` policy. In v8 it was **replaced by `ConcurrencyLimiter`**, which is a specialized rate limiter. The v8 migration guide states explicitly: *"In v8, it's not separately exposed because it's essentially a specialized type of rate limiter: the `ConcurrencyLimiter`."*

Implementation uses `Polly.RateLimiting` (backed by `System.Threading.RateLimiting`):

```csharp
var pipeline = new ResiliencePipelineBuilder()
    .AddConcurrencyLimiter(
        permitLimit: 20,  // max concurrent executions
        queueLimit: 5)    // queue beyond that; then reject
    .Build();
```

When the limit is reached and the queue is full, the strategy throws `RateLimiterRejectedException`. Sizing heuristic: measure p99 latency of the downstream × target rps × 1.2 headroom.

Deep dive: [Bulkhead Pattern](../15-reliability-and-sre/03-bulkhead-pattern.md)

</details>

---

### 5. What is the cascading timeout rule and why does it matter?

<details>
<summary>Reveal answer</summary>

**Each hop's timeout must be shorter than its caller's.** If user → gateway → A → B → DB all have independent timeouts, and the DB timeout (30 s) exceeds A's timeout (10 s), then A gives up before the DB has finished — the work is wasted, and retries amplify the waste.

Senior solution: **deadline propagation**. Pass a `CancellationToken` linked to the caller's remaining time budget. gRPC does this automatically (the call carries a deadline; intermediate hops forward the remaining time). Plain HTTP requires you to do it manually via `CancellationTokenSource`.

Also: `HttpClient.Timeout` default is **100 seconds** — far too long for interactive paths. Set a conservative outer bound there, and use a per-request `CancellationToken` for the real limit. Microsoft's docs: *"only the shorter of the two timeouts will apply."*

Deep dive: [Timeouts](../15-reliability-and-sre/04-timeouts.md)

</details>

---

### 6. A gRPC client sets a deadline of 2 s. The server takes 2.5 s to respond. What does the server observe vs the client?

<details>
<summary>Reveal answer</summary>

The official gRPC blog spells this out: *"both the client and server make their own independent and local determination about whether the remote procedure call was successful."*

- **Client**: receives `StatusCode.DeadlineExceeded` at 2 s; from the client's view the call failed.
- **Server**: may have completed the work and sent the response successfully. The server's view is "success."

Consequence: **idempotency still matters even with deadlines**. A payment call that "failed" at the client can still have debited the customer. Always use idempotency keys for mutating operations across a network, regardless of how reliable your timeouts are.

Deep dive: [Timeouts](../15-reliability-and-sre/04-timeouts.md)

</details>

---

### 7. Name the five Principles of Chaos (from principlesofchaos.org) and explain "minimize blast radius."

<details>
<summary>Reveal answer</summary>

The five advanced principles:

1. **Build a Hypothesis around Steady State Behavior** — measurable system-level metric, predicted before injection.
2. **Vary Real-world Events** — simulate things that actually happen (zone loss, slow disk, DNS flap).
3. **Run Experiments in Production** — staging is a lie about production.
4. **Automate Experiments to Run Continuously** — manual chaos runs once; automated runs daily.
5. **Minimize Blast Radius** — start with the smallest slice that can prove the hypothesis; expand as confidence grows.

*Minimize blast radius* is the safety clause that makes *run in production* acceptable: a single canary pod, a 1% traffic shadow, a single zone — not the entire fleet. AWS FIS enforces this structurally via **stop conditions** tied to CloudWatch alarms: the experiment aborts automatically if blast radius expands.

Deep dive: [Chaos Engineering](../15-reliability-and-sre/05-chaos-engineering.md)

</details>

---

### 8. Describe the four roles in the Incident Command System and why they must be separate people for large incidents.

<details>
<summary>Reveal answer</summary>

Google SRE's ICS (from *Managing Incidents*):

| Role | Job |
|---|---|
| **IC — Incident Commander** | Holds state, delegates, runs the bridge. Does **not** fix. |
| **Ops Lead** | The only person touching the system. *"The operations team should be the only group modifying the system during an incident."* |
| **Comms Lead** | Status page, exec updates, customer comms. Keeps the living incident doc current. |
| **Planning Lead** | Long-horizon: bug tracking, resource requests, handoffs across time zones. |

Why separate? The failure mode is **freelancing** — two engineers independently rolling back while a third pushes a hotfix, conversations spread across five channels, nobody tracks what's been tried. Separating the roles forces serialization through the IC and keeps changes single-threaded through Ops.

Small incident: one person can be IC + Comms. Big incident (cross-team, customers watching): must be separate humans.

Deep dive: [Incident Response](../15-reliability-and-sre/06-incident-response.md)

</details>

---

### 9. What makes a postmortem "blameless" without becoming accountability-free?

<details>
<summary>Reveal answer</summary>

Blameless does **not** mean nameless or consequence-free. It means analysis focuses on **systemic gaps that permitted the mistake**, not the individual who made it. Google SRE workbook: postmortems *"examine gaps in system design that permitted undesirable failure modes."*

Test: does the root-cause section describe a *control that was missing* (no integration test for the real pool, no rollback on error spike) or a *person who slipped up* (Alice deployed the bug)? The first is useful; the second is a dead end.

Accountability still exists:
- Action items have **named owners + due dates + priority**.
- AI completion rates are tracked across incidents; chronically unclosed AIs are a team problem escalated to leadership.
- Service owners sign off on postmortems.

The rule is: **assume the human acted reasonably given what they knew**. Then ask why the system let their reasonable action break production. That question produces changes; "Alice should be more careful" does not.

Deep dive: [Postmortems](../15-reliability-and-sre/07-postmortems.md)

</details>

---

### 10. Why is "5 Whys" a weak root-cause tool, and what do you use instead?

<details>
<summary>Reveal answer</summary>

5 Whys feels rigorous but has three structural weaknesses:

1. **Forces a single causal chain.** Real incidents have multiple simultaneous causes — a failed test *and* a broken alert *and* a manual deploy bypass. 5 Whys picks one and hides the others.
2. **Stops at the human-shaped answer.** *"Why did the deploy fail? Because Alice pushed it."* — the ladder breaks exactly where the systemic question starts.
3. **Confirmation bias.** The first plausible 5-step chain feels "deep enough" and is accepted.

Better:
- **Causal diagram / fault tree** — draw multiple contributing chains converging on the outcome.
- **STAMP / CAST** (Nancy Leveson) — treat the system as a control loop; ask which safety controls were missing, inadequate, or disabled.
- **Separate "root cause" and "contributing factors" sections** in the postmortem template. This alone forces breadth.

At minimum, if you use 5 Whys, run it **for each contributing factor independently** and reconcile. One chain is never enough.

Deep dive: [Postmortems](../15-reliability-and-sre/07-postmortems.md)

</details>

---

### 11. Given SLO = 99.9% over 28 days and current error rate = 1%, what is the burn rate and what do you do?

<details>
<summary>Reveal answer</summary>

Budget = `1 - SLO = 0.1%`. Current rate = 1%. Burn rate = `1.0% / 0.1% = 10`. At that rate, a 28-day budget is exhausted in **2.8 days**.

Action depends on the budget-policy tier:
- **Page, not ticket.** Multi-window multi-burn-rate alerts (Google SRE) typically page at ≈14× for short windows, ≈6× for longer ones — 10× triggers a short-window page.
- **Announce the burn** — budget policy may already require a feature freeze once the remaining budget crosses a red threshold.
- **Diagnose fast** — is this a real outage, a deploy regression, a single misbehaving client? Mitigate the most likely cause (rollback, scale out, failover).
- **Stop experiments** — chaos runs, risky migrations pause until burn rate returns to baseline.

The policy decisions are **pre-written, not argued mid-incident**. Arguing policy in the war room is how freezes get overridden.

Deep dive: [Error Budgets](../15-reliability-and-sre/08-error-budgets.md)

</details>

---

### 12. Product wants to ship a risky feature next sprint. Reliability is at 75% of the error budget consumed. How do you navigate?

<details>
<summary>Reveal answer</summary>

The error-budget policy is the tool. You do not negotiate the facts; you apply the pre-agreed policy.

Example standard policy:
- **Green (>50% remaining)**: launches allowed.
- **Yellow (25–50%)**: launches allowed, no large-surface migrations, reliability prioritized alongside features.
- **Red (<25%)**: freeze non-critical launches; sprint goes to reliability.

At 75% consumed → 25% remaining → **Red**. The risky feature does not ship this sprint.

Frame it to Product:
1. **State the policy** — *"per our error-budget policy, we're in Red. Non-critical launches are paused."*
2. **Anchor on shared interest** — Product wants velocity **sustainably**. Burning the last of the budget makes the next three sprints worse.
3. **Offer a path** — *"if we close AI-7 and AI-12 from last month's incidents, we project recovery by Thursday. Launch slides by one sprint."*
4. **Escalate only on disagreement** — the policy was signed by leadership. If Product wants to override, they go to the same leadership that signed it. Do not override silently; override in writing.

The senior point: **freezes that get overridden once will always be overridden**. Either hold the policy, or update it — never both.

Deep dive: [Error Budgets](../15-reliability-and-sre/08-error-budgets.md) and [Incident Response](../15-reliability-and-sre/06-incident-response.md)

</details>

---

[Back to index](README.md)
