# Soft Skills

> Read the questions, think about your answer, then click to reveal.

---

### 1. What makes a requirement "good"? Give the attributes and an example of each failing.

<details>
<summary>Reveal answer</summary>

A good requirement is necessary, unambiguous, complete, consistent, verifiable, traceable, and feasible.

Failures:
- *"The report should load fast"* — **ambiguous** (fast how? at what concurrency?).
- *"The system must support bulk import"* with no format/size limit — **incomplete**.
- *"Store PII indefinitely"* + *"comply with GDPR"* — **inconsistent** (right-to-be-forgotten).
- *"The UI should be intuitive"* — **unverifiable**.
- A requirement with no ID and no link to a story — **untraceable**.

Deep dive: [Software Requirements Fundamentals](../16-soft-skills/01-software-requirements-fundamentals.md)

</details>

---

### 2. What's the difference between Functional, Non-Functional, and Business requirements?

<details>
<summary>Reveal answer</summary>

- **Business**: *why* — strategic outcome. *"Reduce cart abandonment by 10% this quarter."*
- **Functional (FR)**: *what* — the capability. *"The user can apply a promo code at checkout."*
- **Non-Functional (NFR)**: *how well* — the quality constraints. *"Checkout p95 ≤ 300 ms at 1 000 rps; 99.95% monthly availability."*

A business goal decomposes into FRs and NFRs. NFRs are the ones that shape the architecture — they're not optional. If you don't elicit NFRs, you'll discover them in production.

Deep dive: [Software Requirements Fundamentals](../16-soft-skills/01-software-requirements-fundamentals.md) and [NFR-Driven Architecture](../06-architecture-and-patterns/18-nfr-driven-architecture.md)

</details>

---

### 3. A stakeholder gives you a vague requirement: *"the export needs to be fast"*. Walk me through how you'd refine it.

<details>
<summary>Reveal answer</summary>

Never accept vague NFRs. Force numbers and conditions:

1. *"Fast" relative to what?* Current time, user expectation, competitor?
2. *"How much data?"* Typical size, largest expected size, 95th-percentile size.
3. *"From where? For whom?"* Same region, global, mobile network, desktop?
4. *"Concurrency?"* How many users running exports simultaneously?
5. *"What happens if it's slow?"* Cancel, queue, show progress?

Output looks like: *"Export p95 ≤ 10 s for up to 100 000 rows, for users on broadband, at up to 10 concurrent exports. Longer exports go to a background job with an email notification."*

That's testable, design-guiding, and unambiguous.

Deep dive: [Software Requirements Fundamentals](../16-soft-skills/01-software-requirements-fundamentals.md) and [Requirements Elicitation](../16-soft-skills/02-requirements-elicitation.md)

</details>

---

### 4. When do you use an interview vs a survey vs observation?

<details>
<summary>Reveal answer</summary>

Match the technique to what you're trying to learn:

- **Interview**: depth from a few key informants. Executives, POs, domain experts. Ask open-ended, ask about the *last time* something happened (specifics beat generalities).
- **Survey**: volume across a large population. Quantifies how many users hit a problem; bad at *why*. Combine with interviews of the extremes.
- **Observation / shadowing**: operational workflows. People describe what they *should* do; you watch what they *actually* do. The delta is gold — workarounds, shadow IT, and skipped steps are requirements.

Senior signal: **combining techniques**. Observe → interview → event-storm → prototype → sign off. Single-technique elicitation misses layers.

Deep dive: [Requirements Elicitation](../16-soft-skills/02-requirements-elicitation.md)

</details>

---

### 5. What is Event Storming and when would you run one?

<details>
<summary>Reveal answer</summary>

A facilitated workshop (Alberto Brandolini) where domain experts + engineers map a business process using sticky notes — **orange for domain events** (`OrderPlaced`, `PaymentCaptured`) in chronological order, then **commands** (blue), **aggregates** (yellow), **policies** (purple), **read models** (green), **external systems** (pink), and **hotspots/questions** (red).

Run it when:
- Modeling a complex new domain.
- Trying to discover bounded contexts.
- Unsure where invariants live.
- You need a shared understanding across business and engineering fast.

Not appropriate for CRUD apps or isolated small features — the setup cost isn't recouped.

Deep dive: [Requirements Elicitation](../16-soft-skills/02-requirements-elicitation.md) and [DDD](../06-architecture-and-patterns/13-ddd.md)

</details>

---

### 6. How do you handle conflicting stakeholders (e.g., Security wants detailed audit logs; UX wants sub-100 ms page loads)?

<details>
<summary>Reveal answer</summary>

Don't pretend the conflict doesn't exist. Three steps:

1. **Name it explicitly** — describe both concerns in the stakeholders' own words.
2. **Surface to decision-makers** — the team should not choose between Security and UX silently. Bring data: what does each option cost?
3. **Design around it** — often both can be satisfied with compensating design (async audit writes via queue, sampled logging, CDC to an audit store). Record the chosen trade-off as an ADR.

Use the Mendelow matrix to map who has power + interest. Manage Security closely if they can block release; keep UX informed if they can't. The senior job is to make trade-offs **visible**, not to win them.

Deep dive: [Stakeholders](../16-soft-skills/03-stakeholders.md)

</details>

---

### 7. What's the INVEST checklist for user stories?

<details>
<summary>Reveal answer</summary>

Bill Wake, 2003. A good story is:

- **I**ndependent — can be reordered and delivered alone.
- **N**egotiable — not a contract; details are worked out with the team.
- **V**aluable — delivers value to a user or business outcome.
- **E**stimable — clear enough to size.
- **S**mall — fits within one iteration.
- **T**estable — success and failure are observable.

If any fail, split, clarify, or push back. *"Refactor to Mediator"* has no V; *"sign up and verify email and onboard"* breaks I and S.

Deep dive: [User Stories and BDD](../16-soft-skills/04-user-stories-and-bdd.md)

</details>

---

### 8. Write a Gherkin scenario for "the system rejects an order with zero items".

<details>
<summary>Reveal answer</summary>

```gherkin
Feature: Order confirmation
  As a customer
  I want to confirm my order
  So that it is processed and shipped

  Scenario: Cannot confirm an empty order
    Given I have an order in Draft status
    And the order has zero items
    When I try to confirm the order
    Then the confirmation is rejected
    And I see the message "An order must have at least one item to be confirmed"
    And the order status remains "Draft"
```

Format: **Given** initial context, **When** event, **Then** observable outcome. Multiple Givens/Thens are fine (use `And`). The scenario reads top-to-bottom as a single concrete case — not a generic rule.

Deep dive: [User Stories and BDD](../16-soft-skills/04-user-stories-and-bdd.md)

</details>

---

### 9. How do you prioritize a backlog with MoSCoW vs RICE vs WSJF? When is each the right tool?

<details>
<summary>Reveal answer</summary>

- **MoSCoW** (DSDM): Must / Should / Could / Won't this time. Best for a single release with fixed scope pressure — forces the hard *"this doesn't ship this time"* conversation.
- **RICE** (Intercom): `(Reach × Impact × Confidence) / Effort`. Best for a continuous product backlog — makes **confidence** a first-class dimension.
- **WSJF** (SAFe): `Cost of Delay / Job Size`. Best for portfolio-scale lean/SAFe shops that want to quantify delay cost explicitly.

Common misuse of all three: pseudo-quantifying opinions. Treat the number as a discussion prompt, not a verdict. The best framework is the one the team actually uses.

Deep dive: [Requirements Management](../16-soft-skills/05-requirements-management.md)

</details>

---

### 10. Explain "Choose Boring Technology" and innovation tokens.

<details>
<summary>Reveal answer</summary>

Dan McKinley's essay: the short-term wins of new tech rarely outweigh the long-term operational cost. Companies have a finite number of **innovation tokens** — roughly three in an early-stage company. Each new technology choice (database, queue, language, deployment platform) spends one.

The discipline: **spend tokens on the parts that actually differentiate the business**. Everything else should be boring — known failure modes, large community, easy to hire for. "Boring" doesn't mean obsolete; it means proven.

Applied to an interview: when asked *"would you use Kafka for this?"*, the good answer starts with *"probably not — what's wrong with Service Bus / RabbitMQ that we already run?"* and earns the novel choice only if the problem demands it.

Deep dive: [Technology Selection](../16-soft-skills/07-technology-selection.md)

</details>

---

### 11. How do you adapt the same message for an executive vs an engineer?

<details>
<summary>Reveal answer</summary>

Same truth, four audiences — **lead with what each can act on**:

- **Executive**: outcome-first, one-liner, risk/cost. *"Payments slip by 4 weeks; no revenue impact; we're moving one engineer."*
- **Product**: scope and sequencing. *"We'll land Payments MVP mid-Q3; Catalog search ships as planned."*
- **Engineer**: trade-offs, rationale, references. *"Gateway lacks refund webhook idempotency; implementing outbox against duplicate callbacks (ADR-0019). PR #234."*
- **Ops/SRE**: runbooks, dashboards, on-call impact. *"Two new workers, new queue `pay.outbox`; runbook in PR."*

BLUF (Bottom Line Up Front), short paragraphs, concrete examples, name the trade-off. The junior failure is writing once and expecting the reader to translate.

Deep dive: [Communication and Audience](../16-soft-skills/06-communication-and-audience.md)

</details>

---

### 12. What are the main anti-patterns in stakeholder/requirements work?

<details>
<summary>Reveal answer</summary>

- **Verbal-only decisions** — *"we agreed in the meeting"* evaporates in weeks.
- **Priority by loudness** — the insistent stakeholder wins, the strategic one starves.
- **Frozen requirements** — the world changes; re-baseline at milestones.
- **Stakeholder by title only** — the real blocker may not be the named approver.
- **Treating NFRs as nice-to-have** — they shape the architecture; they're not optional.
- **Designing the solution in the interview** — *"would a button help?"* anchors the answer. Ask about the problem.
- **"Consent equals enthusiasm"** — a stakeholder saying *"sure"* in a crowded meeting is not approval. Confirm in writing.

Deep dive: [Stakeholders](../16-soft-skills/03-stakeholders.md) and [Requirements Management](../16-soft-skills/05-requirements-management.md)

</details>

---

[Back to index](README.md)
