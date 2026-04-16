# Communication and Audience

A technically correct solution that isn't understood is, in practice, not adopted. Senior engineers spend a significant fraction of their time **communicating** — docs, diagrams, reviews, messages, meetings. The craft is **adapting the same information** to the audience that needs it.

## The four audiences you'll keep meeting

| Audience | Cares about | Typical format |
|---|---|---|
| **Executives / Sponsors** | Outcomes, risk, cost, timeline | 1-slide summary, 3-line status, quarterly review |
| **Product / PMs / Designers** | User impact, scope, what's shippable when | Feature briefs, demos, release notes, roadmap |
| **Engineers (same team / other teams)** | Trade-offs, design, interfaces, rationale | RFCs, ADRs, PR descriptions, architecture diagrams |
| **Operations / Support / SRE** | How to run it, how to fix it, what to watch | Runbooks, dashboards, on-call guides, postmortems |

Your job is to carry the *same underlying truth* across all four, in the language each one speaks. The failure mode — the one that marks junior/mid communication — is writing for yourself and expecting the reader to translate.

## What changes by audience

Take one statement and watch it adapt:

> **Underlying truth:** *"The payments service will not be ready by Q2 because the gateway integration is taking three times the estimated effort; we are reallocating one engineer from the catalog service."*

**To an executive (CFO):**
> "Payments slip by 4 weeks to mid-Q3. Root cause: gateway integration is more complex than estimated. We're reallocating one engineer from Catalog, which stays on track. No revenue impact before end of Q3."

**To product:**
> "We're pulling an engineer onto Payments to recover 3 weeks of the slip. Catalog keeps its scope but won't pick up the new filter work this sprint — we'll re-plan in two weeks when the gateway PoC lands."

**To engineering:**
> "PSP integration estimate was wrong — the sandbox doesn't cover webhooks for refunds, so we're hand-rolling idempotency against duplicate callbacks (see ADR-0019). I'm pairing with A. on that; Catalog search refactor is deprioritized. PR #234 has the outbox change."

**To operations:**
> "No operational change yet. When Payments ships, expect a new queue `pay.outbox` and a worker with 2× replicas. Dashboards and runbook drafts in the PR; review meeting next Tuesday."

Same truth, four messages. **Lead with what each audience can act on.**

## Writing principles that keep showing up

### 1. Lead with the answer (BLUF — Bottom Line Up Front)

Start with the conclusion. Details follow for those who want them. The inverted pyramid that newspapers use is the right shape for engineering docs: the reader who reads only the title + first paragraph should still know the decision.

> **Bad:** *"We considered Option A, which has advantages X and disadvantages Y, and Option B, which has…"* — the reader reaches the decision on page 3.
>
> **Good:** *"We chose Option B. Rationale and alternatives below."*

### 2. Show the why, not just the what

Code shows *what*. The doc exists because the *why* isn't in the code. Every architectural doc, ADR, and RFC should answer *why this, why not the alternatives, what does this cost us*. See [Design Docs, C4 and ADRs](../06-architecture-and-patterns/17-design-docs-c4-adr.md).

### 3. Write to be skimmed

Most readers don't read — they scan. Use:

- **Headings** that describe what's under them, not just *"Section 2"*.
- **Short paragraphs** (3–5 lines max in technical docs).
- **Bullets and tables** for lists, not prose.
- **Bolded key terms** sparingly — if everything is bold, nothing is.
- **TL;DR** at the top of anything over two pages.

### 4. Reduce cognitive load

Pick the level of abstraction the audience needs and stay there. A diagram mixing users, HTTP endpoints, DB tables, **and** internal classes asks the reader to re-zoom on every line. Split it. C4 levels make this explicit (see [UML Diagrams](../06-architecture-and-patterns/19-uml-diagrams.md)).

### 5. Use concrete examples

*"The system is event-driven"* is abstract. *"When a customer places an order, the Orders service publishes `OrderPlaced`, and the Shipping service picks it up within ~200 ms and creates a shipment record"* is concrete. Prefer the second.

### 6. Name the trade-off

Every engineering decision has a downside. Naming it up front (*"this adds ~50 ms to the write path, in exchange for strong consistency"*) builds trust; hiding it makes you the opposition's best argument when it shows up in production.

## Diagrams — when and what

Diagrams are the fastest communication tool. Rules of thumb:

- **One diagram, one question.** *How does money flow?* *What are the order states?* *Who owns the data?* Trying to answer all three in one diagram produces an unreadable one.
- **Always a legend.** Shapes and arrows have meaning; don't make the reader guess.
- **Text-based over image-only.** Mermaid, PlantUML, and Structurizr live in Git and get reviewed like code. A `.png` drifts silently.
- **C4 for system shape**, sequence for flows, state for lifecycles, ER for schemas. See [UML Diagrams](../06-architecture-and-patterns/19-uml-diagrams.md).

## PR descriptions and commit messages

The most frequent communication a senior does. Minimum bar:

**Commit message:**
```
orders: outbox-based publishing for OrderConfirmed

Replace the post-commit publish call with an outbox row written in
the same transaction, published asynchronously by a relay. Removes the
dual-write failure where the order was committed but the event lost.

Ref: ADR-0019, ORD-1234
```

**PR description:**
- What changed and why (2–4 sentences).
- What didn't change (sets reviewer expectations).
- How you verified it works.
- Rollback plan if non-trivial.
- Links to the story, ADR, or RFC this implements.

A PR reviewer shouldn't have to read the whole diff to know what the PR is about.

## Meetings — the 2-sentence test

Before scheduling, write the one sentence of purpose and the one sentence of expected outcome. If you can't, don't hold the meeting — send a written update.

> *"We're deciding whether to use PostgreSQL or Mongo for the Orders service. By the end, we have a written decision in ADR-0007."*

If the meeting's goal is *"sync"* or *"status"*, it belongs in a text channel.

## Feedback — giving and receiving

As a senior you give and receive a lot of review. Both are communication.

- **Be specific.** *"This looks wrong"* is not feedback. *"This loops N times over the full list; consider a Dictionary"* is.
- **Attack the idea, not the person.** *"I don't see how this handles the partial-failure case"* beats *"you didn't think about failure"*.
- **Recognize the asymmetry.** Writing takes an hour; a two-word review (*"approved"*) takes a minute. Be the reviewer you want when you're the author.
- **Receive it well.** If you argue every piece of feedback, people stop giving it. You don't have to implement every suggestion, but you should read every one of them charitably.

## Active listening

Half of communication is listening. In interviews, reviews, and stakeholder conversations:

- **Let the other person finish.** Cutting them off loses information.
- **Paraphrase back.** *"What I'm hearing is X — is that right?"* catches misinterpretations in the moment.
- **Separate the problem from the proposed solution.** Stakeholders often arrive with a solution; your job is to ask *why* until you find the problem underneath.

## Pitfalls

- **Writing for yourself.** If the doc only makes sense if you already know the context, it's worthless to anyone onboarding.
- **Over-explaining to executives.** They don't want the 3-page context; they want the decision and the risk.
- **Under-explaining to engineers.** "It works, trust me" doesn't survive a month.
- **Using different jargon in different docs.** If the ADR calls it *Order*, the PR calls it *Purchase*, and the runbook calls it *Transaction*, you built a glossary problem.
- **Silent confusion.** If you don't understand what a stakeholder meant, ask. Nodding through a misunderstanding costs more than the awkwardness.
- **Emotional messages in writing.** Heated Slack threads and email are how disagreements escalate. Move them to a call; summarize the decision in writing afterward.

## Rule of thumb

> Every artifact — message, doc, diagram, PR, meeting — answers the question *"what does this audience need to do after reading this?"*. If there's no action or decision it enables, it probably shouldn't exist.

---

[← Previous: Requirements Management](05-requirements-management.md) | [Back to section](README.md) | [Next: Technology Selection →](07-technology-selection.md)
