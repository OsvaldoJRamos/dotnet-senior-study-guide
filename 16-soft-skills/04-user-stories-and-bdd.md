# User Stories and BDD

A **user story** is a short description of a requirement written from the perspective of someone who wants something from the system. **BDD** (Behavior-Driven Development) extends stories with a concrete, executable format for their acceptance criteria. Together they are the agile-standard way of describing what to build.

## User stories — the standard form

Mike Cohn popularized the three-line template (*User Stories Applied*, 2004):

```
As a [role / persona],
I want [capability],
so that [benefit / outcome].
```

Example:

```
As a customer,
I want to save my delivery addresses,
so that I don't have to retype them on every order.
```

The first two lines describe **what and for whom**. The third line is the one people skip — and it's the most important. Without *so that*, you're shipping a feature with no testable outcome.

## INVEST — the checklist for a good story

Bill Wake, 2003 (`agilealliance.org/glossary/invest/`). A story should be:

| Letter | Meaning |
|---|---|
| **I** | **Independent** of other stories (can be reordered, delivered alone) |
| **N** | **Negotiable** — not a contract; details are worked out collaboratively |
| **V** | **Valuable** — delivers value to a user or business outcome |
| **E** | **Estimable** — clear enough that the team can size it |
| **S** | **Small** — fits inside a single iteration |
| **T** | **Testable** — success and failure are observable |

If a story fails any of these, split it, clarify it, or push back before committing.

### Common story smells

- **Technical-only story.** *"Refactor the checkout service to use Mediator."* Who benefits? If there's no user/outcome, it's a task, not a story — put it in the technical backlog.
- **"And" in the title.** *"Sign up AND verify email AND onboard"* is three stories.
- **Novel-length acceptance criteria.** If the story needs 20 AC lines, split it.
- **Dependency tangle.** *"Blocked by story X and Y."* If it can't ship alone, the slice is wrong.
- **"As a user" without a persona.** Every user is the same? Probably not — different personas often have different flows.

## Acceptance criteria

Acceptance criteria (AC) turn a story into something QA and engineering can **verify**. Two dominant styles:

### 1. Simple checklist

```
- Customer can view a list of saved addresses on the account page.
- Customer can mark one address as default.
- Customer can delete a saved address (not deletable if used in an open order).
- Save fails with a clear error if the address is incomplete.
```

Works well for straightforward CRUD. Less good at capturing state-dependent behavior.

### 2. BDD / Gherkin (Given-When-Then)

Dan North introduced BDD in 2003 as an evolution of TDD, framed in the language of the domain. The canonical format:

```
Given [initial context]
When [event occurs]
Then [ensure some outcomes]
```

Real example:

```gherkin
Feature: Save delivery address
  As a customer
  I want to save my delivery addresses
  So that I don't have to retype them

  Scenario: Save a new valid address
    Given I am logged in
    And I have no saved addresses
    When I submit a new address with all required fields
    Then the address appears in my saved addresses list
    And the address is marked as default

  Scenario: Reject incomplete address
    Given I am logged in
    When I submit an address missing the postal code
    Then the save is rejected
    And I see the message "Postal code is required"

  Scenario: Cannot delete address used by an open order
    Given I have an address used by an open order
    When I try to delete that address
    Then the delete is rejected
    And I see the message "Address is in use by order #{orderId}"
```

### Why Gherkin

- **Readable by non-engineers.** Product, QA, and business can review and sign off on the same document the team builds against.
- **Close to test code.** Tools like **SpecFlow** (.NET), **Cucumber** (many languages), and **Reqnroll** (SpecFlow's successor) execute Gherkin as tests. A scenario written for alignment becomes the regression suite.
- **Forces concrete thinking.** Ambiguity in the story shows up as *"wait, what does step 2 actually mean?"* at the whiteboard, not at QA time.

### Where Gherkin is overkill

Not every backlog item needs Gherkin. A UI pixel tweak or a dependency bump doesn't benefit from a scenario. Use it when:

- The behavior has **state-dependent branches** (multiple Givens lead to different outcomes).
- **Non-engineers** must review or sign off on the behavior.
- You want the AC to double as **automated tests**.

## The three amigos

A cheap practice before committing to any non-trivial story: **product + engineering + QA** review the AC together for 15 minutes.

- Product catches misunderstood business rules.
- Engineering catches technical conflicts and edge cases.
- QA catches untestable ACs and missing negative paths.

If a story survives a three-amigos pass, most of the ambiguity is gone before work starts. The cost is dwarfed by the rework it prevents.

## Story splitting

When a story is too big, split along axes that each deliver value:

| Axis | Example |
|---|---|
| **Workflow steps** | Ship step 1 of 3 first; 2 and 3 follow |
| **Happy path first** | Ship the 80% case; add edge cases next |
| **Data types** | Support domestic addresses first; international next |
| **Interface** | API only first; UI next |
| **Rules / variation** | One payment method first; the rest incrementally |
| **CRUD** | Create + read first; update + delete next |

Split to preserve **value per slice**. A slice that delivers nothing until the next slice arrives isn't a split — it's just a phase.

## Estimating

Story points, T-shirt sizes, or ideal days — the estimation unit matters less than consistency and honesty. What matters:

- Estimate as a team, not as individuals.
- Anchor on a **reference story** (*"this is about the size of Story X from last sprint"*).
- If estimates spread widely, the story is under-specified. Re-refine before committing.
- No-estimate methods (#NoEstimates, counting stories, flow metrics) work when the team has stable story sizes and a mature domain. They are not an excuse to skip refinement.

## Pitfalls

- **Story theater.** Rewriting tasks as "As an engineer, I want to refactor…" to look agile. It fools nobody and dilutes the real stories.
- **AC as implementation spec.** *"Use `IAuthService.Refresh()`"* is implementation, not acceptance. AC describes behavior; design describes how.
- **"Non-negotiable" stories.** The **N** in INVEST is negotiable. Product decides what and when; engineering decides how and at what cost. A story handed down with all of those pre-decided is a spec.
- **Gherkin as documentation theater.** Scenarios written to *look* thorough but not run as tests rot fast. If the feature file isn't executed by CI, it's a stale Word doc.
- **Ignoring non-functional criteria.** A story can have AC on performance (*"p95 ≤ 200 ms"*), accessibility, or auditability. If the behavior has an NFR, write it.

## Rule of thumb

> A good user story is small enough to ship in a week, clear enough that two engineers build the same thing, and testable enough that *done* is unambiguous. Anything else is a wish.

---

[← Previous: Stakeholders](03-stakeholders.md) | [Back to section](README.md) | [Next: Requirements Management →](05-requirements-management.md)
