# Technology Selection

Picking a framework, database, language, or platform is one of the highest-leverage decisions a senior engineer makes — it binds the team for years and is much harder to reverse than any line of code. The anti-pattern is picking by **novelty**; the discipline is picking by **fit + operational cost**.

## Start here: Choose Boring Technology

Dan McKinley's essay (`boringtechnology.club`) is the best one-page framing of the problem. The thesis:

> *"The new tech is going to make our work so much easier in the near term that this benefit outweighs the cost of dealing with that technology indefinitely into the future"* — but in practice, operational costs of new technologies typically exceed their short-term benefits.

### Innovation Tokens

McKinley's mental model: a company has **a small, finite number of innovation tokens** — roughly three in an early-stage company. Each new technology choice spends one. Spend them on the parts of the system that are actually a differentiator for the business; use boring, proven tools for everything else.

> If your business goal already demands tokens (a novel ML model, a new market), you can't also afford to innovate on your database, queue, and deployment tool. Something has to stay boring.

"Boring" here doesn't mean *bad* or *obsolete* — it means *"the failure modes are well-known, the community is large, and hiring for it is easy"*. PostgreSQL is boring. Kubernetes, on a team that already runs it, is boring. A new Rust async runtime you heard about on HN is not.

## The evaluation checklist

When you're considering adopting a new tool — a framework, database, language, messaging system, cloud service — walk through this list **in writing**, ideally as an [ADR](../06-architecture-and-patterns/17-design-docs-c4-adr.md).

### 1. Fit to the problem

Does it actually solve the problem, or does it solve an adjacent one you'll have to work around?

- What access patterns / workloads does it optimize for?
- What assumptions does it make about the shape of your data, your team, your scale?
- Have other teams with your shape used it successfully at production scale?

### 2. Team maturity and ramp cost

- How many people on the team have real experience with it? (Not conference talks — production.)
- What's the realistic learning curve — weeks or months?
- Will you need new hires to staff around it? Is the labor market for it healthy?
- Does adopting it cross a cultural line — sync → async, imperative → functional, etc.?

### 3. Ecosystem and community

- Is there a large community answering questions on Stack Overflow / Discord / GitHub?
- How many third-party libraries exist around it? Are they maintained?
- How many production postmortems have been published about it — and are the common failure modes known?
- Is there enterprise support available if you need it?

### 4. Maintenance and release cadence

- Is there a clear **LTS** (Long-Term Support) story, or will you be dragged onto every minor release?
- What's the deprecation pace? A framework that rewrites its core API every 18 months is a tax.
- Is it maintained by an active, funded org or a single maintainer?
- What's the GitHub signal — commits in the last 30 days, issue response time, open/closed PR ratio?

### 5. Operational cost

This is the one juniors miss. Adopting a technology costs more than installing it.

- **Observability**: can you plug it into your existing metrics / logs / traces? If it needs its own stack, that's another system to run.
- **Backup / restore** (for stores): tested, documented, automated?
- **Upgrade path**: how do you go from v3 → v4 without downtime?
- **Security posture**: auth model, secrets management, CVE cadence.
- **On-call burden**: who gets paged when it misbehaves, and do they know what to do?
- **Cost at scale**: pricing curve — does it stay sane at 10× your current size?

### 6. Lock-in

Every technology is some lock-in; the question is how painful an exit would be.

- How portable is the code around it — would changing it require a rewrite or a swap?
- Is the data format proprietary or open?
- Is the API a well-known interface (SQL, S3-compatible, OAuth) or vendor-specific?
- Multi-cloud is rarely worth it on purpose; accidental single-cloud lock-in is still a risk to name.

### 7. Long-term outlook

- Is adoption trending up or down? (Google Trends, Stack Overflow survey, JetBrains survey, LinkedIn jobs.)
- Is the vendor financially healthy, if it's a commercial product?
- Is there a credible successor technology that will replace it in 3–5 years?

## A useful rubric

Score each candidate on the seven dimensions above (0–3 each). The highest score doesn't automatically win — but candidates scoring ≤ 1 on any of "fit", "ops", or "team maturity" are usually disqualified.

| Dimension | Candidate A | Candidate B |
|---|---|---|
| Fit to problem | 3 | 2 |
| Team maturity | 3 | 1 |
| Ecosystem | 2 | 3 |
| Maintenance / LTS | 3 | 2 |
| Ops cost | 2 | 1 |
| Lock-in | 2 | 2 |
| Long-term outlook | 3 | 3 |

Here A wins on team maturity and ops, even though B scores slightly better on ecosystem. The rubric is a **discussion prompt**, not a scoreboard — its value is forcing every dimension to be considered explicitly.

## A practical process

1. **Name the problem**, not the solution. *"We need durable async processing with at-least-once delivery"*, not *"we need Kafka"*.
2. **Enumerate 3 candidates.** One of them is always "do nothing" / "extend what we have" — often wins.
3. **Build a small PoC** for the top 2, if scale / ops matters. A weekend spike catches more than weeks of spec reading.
4. **Write the ADR.** The exercise itself catches half the problems.
5. **Pilot in one bounded context.** Don't roll out org-wide on day one.
6. **Monitor the real cost for 3 months** — ramp time, incidents, upgrade pain. If it's worse than predicted, own the decision and correct it.

## Special cases

### Preview / alpha / 0.x versions

Can rename types between minor versions. Acceptable for exploration; unacceptable for production dependencies unless you own the cost of chasing breaking changes. For fast-moving preview SDKs, always verify against the current README and release notes before writing code — see the CLAUDE.md Source Verification rules.

### Free / OSS vs commercial

OSS is not free — it's paid in your team's time. Commercial is not expensive — compared to an engineer's fully-loaded cost, licenses are usually rounding error. Evaluate **total cost of ownership**, not the license line.

### Organization mandates

Some choices are not yours to make — corporate standards, approved vendor lists, cloud exclusivity. Work within them. If the mandate is actively harmful, the lobbying happens through ADRs and quantified impact, not by quietly working around it.

## Pitfalls

- **Resume-driven development.** Picking a tool because "I want to learn it" or "it looks good on a résumé" costs the team years.
- **Best tool for each job.** Taken literally, ends in polyglot hell. See [Polyglot Persistence](../09-data-access/09-polyglot-persistence.md) for the cost of every extra store.
- **Trusting benchmarks.** Public benchmarks are almost always optimized for the benchmark, not your workload.
- **Ignoring the team.** A technology nobody on the team knows will be slower for 6 months than the "inferior" one they already know.
- **Under-valuing stability.** A boring tool that hasn't had a breaking change in 5 years is doing its job — not failing to innovate.
- **Single-vendor lock-in without eyes open.** Being on AWS, Azure, or .NET is usually the right call — but know the exit story if the pricing/terms change.

## Rule of thumb

> Save your **innovation tokens** for the parts of your system that matter to the business. Everything else should be the most boring, well-trodden technology you can get away with.

---

[← Previous: Communication and Audience](06-communication-and-audience.md) | [Back to section](README.md)
