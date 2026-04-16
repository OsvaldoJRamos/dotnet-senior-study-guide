# Requirements Elicitation

**Elicitation** is how you extract what the system actually has to do from people who often don't know themselves — stakeholders, users, experts, operations. Done badly, you build the wrong thing efficiently. Done well, you save weeks of rework.

No single technique covers every situation. The senior skill is matching the technique to the context.

## The core techniques

| Technique | What it is | Best for | Limitations |
|---|---|---|---|
| **Interviews** | 1:1 or small-group conversation with a stakeholder | Deep insight from key informants; early scoping | Time-consuming; biased by interviewer framing |
| **Surveys** | Structured questionnaire to many respondents | Quantitative signal from a large user base | Surfaces what people *say*, not what they *do*; low depth |
| **Observation / shadowing** | Watch users do their real job | Operational workflows; catching implicit knowledge | Hawthorne effect (behavior changes when observed); time-heavy |
| **Brainstorming** | Group session to generate options | Early divergent thinking; feature ideation | Without structure, dominated by loudest voices |
| **Prototyping** | Throwaway low-/mid-fidelity UI to validate ideas | UX validation before committing engineering | Prototype gets mistaken for the real product |
| **Use cases / user stories** | Formalized description of actor-system interactions | Documenting agreed scope; aligning teams | Bureaucratic if overused |
| **Event Storming** | Sticky-note workshop mapping domain events over time | Complex domains; discovering bounded contexts | Needs facilitator and in-person (or high-quality remote) |
| **Document analysis** | Read existing contracts, regulations, manuals | Compliance work; legacy system rewrites | Documents lie; always validate with a human |
| **Workshops** | Structured multi-party sessions | Cross-team integration contracts, kickoffs | Expensive; needs preparation to avoid becoming "a meeting" |

## Interviews — the default technique

Most elicitation starts here. The non-obvious parts:

- **Open-ended first, closed second.** *"Walk me through what happens when a customer returns an item"* before *"Do you refund to the original payment method?"*. Closed questions anchor the answer — open questions surface the actual process.
- **Ask about the **last time** it happened, not in general.** *"Tell me about the last tricky refund you processed"* beats *"How do refunds usually work?"*. People reconstruct specifics accurately; they rationalize generalities.
- **Listen for disagreements.** If two stakeholders describe the same process differently, that's not a communication failure — it's a hidden branch in the process, and you just found a requirement.
- **Record decisions, not just facts.** *"We decided not to support partial refunds because the payment gateway charges a flat fee"* is worth more than *"partial refunds not supported"*.
- **One note-taker, one interviewer.** Trying to do both produces poor notes and missed follow-ups.

## Observation (shadowing)

People describe the process they **should** follow. Observation shows the process they **actually** follow. The delta is where bugs, shadow IT, and workarounds live.

- Ideal for operational domains — warehouses, call centers, trading desks.
- Ask permission explicitly; explain you're here to improve the tool, not to audit performance.
- Watch for *"oh, we always have to paste this back into the spreadsheet because the system doesn't support X"* — those are requirements.
- Combine with interviews: observe first, then discuss what you saw.

## Event Storming (Alberto Brandolini)

A facilitated workshop for complex domains. Participants write **domain events** on orange stickies in chronological order ("OrderPlaced", "PaymentCaptured", "ItemPicked", "ShipmentDelayed"), then layer **commands** (blue), **aggregates** (yellow), **policies** (purple), **read models** (green), and **hotspots** (red — open questions).

Used to:

1. Surface the real process (usually differs from org-chart abstractions).
2. Find bounded contexts — where the language changes, there's a boundary.
3. Discover aggregates and the invariants they protect.
4. Reveal questions early, while everyone who knows the answer is in the room.

Best combined with DDD (see [DDD](../06-architecture-and-patterns/13-ddd.md)).

## Prototyping

Low-fidelity prototypes (Figma, Balsamiq, HTML sketches) are fast and cheap. Their power is that users react to **something specific** instead of abstract questions. The risk is that they get taken as final — communicate loudly that this is throwaway.

Three fidelity levels:

- **Paper / whiteboard** — minutes per iteration; validates whether you understand the problem.
- **Clickable wireframe** — hours per iteration; validates the flow.
- **Interactive mock** — days per iteration; validates the interaction details.

Stop at the lowest fidelity that resolved the question. Going higher steals engineering time.

## Surveys

Great for **volume** (hundreds of users), terrible for **depth**. Use when you need to quantify an opinion across a population — *"what fraction of our users hit this problem?"* — not when you need to understand *why*.

- Pilot with 5 people before sending to 500; ambiguous questions are invisible until someone misreads them.
- Don't ask questions you won't act on. Every answered question is a promise.
- Combine with interviews — use the survey to find extremes, then interview the extremes.

## How to choose the right technique

| Situation | Technique |
|---|---|
| Executive with 30 minutes, strategic scope | Short, structured **interview** focused on outcomes |
| Operational users, complex manual workflow | **Observation** + follow-up interviews |
| New complex domain, cross-functional scope | **Event Storming** + interviews with domain experts |
| Validating UX early, before engineering | Low-fidelity **prototyping** + user testing |
| Gauging adoption risk across thousands of users | **Survey** + interview the tails |
| Cross-team integration contract | **Workshop** + document the contract as an API spec |
| Compliance-driven rewrite (LGPD, SOC 2) | **Document analysis** + interviews with legal/compliance |

In practice you'll **combine** techniques. A realistic engagement: document analysis to learn the current state → 6 interviews → an event-storming workshop → a prototype of the new UX → iteration → signed-off acceptance criteria.

## When to stop eliciting

Elicitation has diminishing returns. Stop when:

- New interviews repeat what earlier ones already said.
- The acceptance criteria for the next slice are unambiguous.
- The risks of building the wrong thing are lower than the cost of further delay.

The opposite failure — **analysis paralysis** — is as expensive as under-eliciting. A week of more interviews never answers the question that two days of prototyping would.

## Pitfalls

- **Designing the solution in the interview.** *"Would a button here help?"* anchors the answer. Ask about the problem, offer the solution later.
- **Listening only to the loud stakeholder.** The quiet ones usually have the sharpest edge cases; the loud ones usually have the strategy. You need both.
- **Trusting the document.** Specs and SOPs describe what should happen; reality is in observation.
- **Not writing anything down.** *"We discussed it in the meeting"* is the fastest way to re-elicit the same requirement three times.
- **Single-source bias.** If every requirement comes from one person, you have a fragile process — when they leave, so does the knowledge.

## Rule of thumb

> Elicit in the **layer** where the truth lives: strategy in exec interviews, intent in PO conversations, actual workflow in observation, domain invariants in event storming, UX acceptance in prototypes. Using the wrong technique for the layer is how you miss requirements you'll pay for later.

---

[← Previous: Software Requirements Fundamentals](01-software-requirements-fundamentals.md) | [Back to section](README.md) | [Next: Stakeholders →](03-stakeholders.md)
