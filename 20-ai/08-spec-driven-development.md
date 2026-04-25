# Spec-Driven Development

When teams adopt AI coding agents (Claude Code, Cursor, GitHub Copilot agents), the bottleneck moves from *typing speed* to *aligned intent*. Spec-Driven Development (SDD) is the practice that emerges to manage that bottleneck at enterprise scale.

## The progression: vibe coding → plan mode → SDD

| Mode | What the human does | What the AI does | Where it breaks |
|---|---|---|---|
| **Vibe coding** | Iterative prompting; "make it do X, fix Y" | Generates code one prompt at a time | Loses context across sessions; rework on every restart |
| **Plan mode** | Asks for a plan, reviews, then approves | Drafts a step-by-step plan, then executes | Plans drift from real intent; team lacks visibility |
| **Spec-Driven Development** | Authors and reviews a durable spec; agent works against it | Sustained autonomous execution against the spec | Specs themselves rot if not maintained |

The InfoQ article on enterprise SDD (linked in the section README) frames this as an evolution of how humans and AI agents collaborate, not a tooling fad.

## What a spec actually is

A spec is **shared context** — not a Word document handed off to an engineer, and not a prompt handed off to an agent. It captures:

- **Intent** — what the change is, in business terms.
- **Constraints** — non-functional requirements (perf, compliance, security).
- **Interfaces** — the contracts the change touches (APIs, schemas, events).
- **Acceptance criteria** — how the change is validated.

A spec lives in version control alongside code. It is updated as the work proceeds. It is the primary unit of work, not a side artefact.

## Multi-phase collaboration

The article identifies three phases, each owned by a different role:

| Phase | Owner | Output |
|---|---|---|
| **Discover** | Product / Domain expert | What needs to change and why (in business terms) |
| **Design** | Architect / Tech lead | How it changes — interfaces, data flow, dependencies |
| **Tasks** | Engineer | How it gets executed — file-by-file or repo-by-repo work units |

This decomposition prevents what the article calls **SpecFall** — the failure mode where one role writes the entire spec, the other roles never engage, and the spec becomes outdated documentation.

## Why this matters at enterprise scale

The article's core argument: *"Teams building execution context through cross-functional specs are superior to individuals optimizing prompts or chasing smarter models."*

Concretely, an enterprise SDD setup needs:

- **Cross-repository orchestration** — a feature often spans 3-5 repos; specs must decompose into per-repo sub-issues.
- **Backlog integration** — specs live next to Jira/Linear tickets, not in a separate AI-only tool. MCP servers are increasingly the integration layer.
- **Brownfield strategy** — most enterprises don't get a greenfield. Specs must work over existing code, not assume a clean slate.
- **Role-specific harnesses** — security agents enforce auth and PII rules, performance agents enforce latency budgets, etc. The spec triggers the right harness for the right phase.

## Anti-patterns

- **Writing the spec after the code.** That's documentation, not SDD. The spec drives the work; if it lags, you're back to vibe coding with extra paperwork.
- **One-person specs.** If only the engineer writes the spec, you've cut Discover and Design out — you're doing solo Plan Mode with extra steps.
- **Specs that describe implementation, not intent.** "Use Redis for caching" is brittle. "Reads must return in <50ms p95" is durable; the engineer (or agent) picks Redis if that fits.
- **Treating every bug as a code patch.** The article's framing: bugs are usually harness or spec gaps. The fix is often to update the spec or the agent's constraints, not just patch the code.

## Where it fits in the .NET / Azure world

SDD is tooling-agnostic, but in the Microsoft stack the practical surfaces today are:

- **Markdown specs in the repo** (`docs/specs/feature-name.md`) — the cheapest starting point.
- **GitHub Issues with structured templates** — for cross-repo decomposition.
- **MCP servers** — connecting the agent to Jira, Linear, internal docs, or the company's design system.
- **Agent harnesses** — `CLAUDE.md`, `.cursor/rules`, custom agents in GitHub Copilot — encoding role-specific constraints.

> **Senior signal:** SDD isn't "use AI better." It's a recognition that AI moves the bottleneck from *writing code* to *aligning humans on what to build* — and that alignment is the work.

---

[← Previous: AI Architecture Scenarios](07-ai-architecture.md) | [Back to index](README.md)
