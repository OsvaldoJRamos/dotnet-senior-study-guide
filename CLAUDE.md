# CLAUDE.md — Project Instructions

## What is this project?

A comprehensive .NET senior study guide — organized markdown files covering C#, ASP.NET Core, architecture, cloud, DevOps, testing, Angular, AI, and interview prep.

**Repository:** https://github.com/OsvaldoJRamos/dotnet-senior-study-guide

## Git Workflow

- **Never commit directly to `main`**. Always create a feature branch and open a Pull Request.
- Branch naming: `docs/<short-description>` (e.g., `docs/add-aws-lambda-topic`)
- PR title should be concise and descriptive
- After creating the PR, share the URL with the user for review

## Language Rules

- **All content must be in English** — headings, explanations, code comments, variable names, class names, string literals
- **Code examples** should use English identifiers: `Order` not `Pedido`, `Customer` not `Cliente`, `name` not `nome`
- **Technical terms** stay in their original form (e.g., SOLID, CQRS, DI)

## File Structure

```
XX-section-name/
├── README.md           # Section index with numbered file list
├── 01-topic-name.md    # Individual topic file
├── 02-another-topic.md
└── ...
```

### Naming conventions

- Directories: `XX-kebab-case-in-english/` (e.g., `01-csharp-fundamentals/`)
- Files: `NN-kebab-case-in-english.md` (e.g., `01-dotnet-ecosystem.md`)
- Numbers are zero-padded: `01`, `02`, ..., `10`, `11`

### Section ordering (MANDATORY)

**Sections must be ordered from the most basic and essential topics to the most complex.** The sequence reads as a learning path: language fundamentals → CS fundamentals → architecture → web → security → data → quality → ops → production concerns → distributed systems → system design → specializations → non-technical → assessment. Never dump a new section at the end by default — find the right spot in the progression.

**The `self-assessment/` section must always be the LAST numbered section.** It aggregates questions for every other section, so it can only exist after the rest of the material is defined.

When adding a new section:
1. Determine where it naturally fits in the basic-to-complex progression (see the "Current Sections" table below for the reference order).
2. Renumber every downstream section (including `self-assessment/`) and update ALL cross-references.
3. Update section README titles (`# XX - Section Name`), the root `README.md`, the CLAUDE.md "Current Sections" table, and any `../XX-section-name/` paths in other files (including `21-self-assessment/`).
4. Verify with `grep` that no stale `XX-section-name` paths remain.

Never insert a new section after self-assessment, and never add content files inside it that belong in a dedicated topic section.

## File Format

Every topic file follows this structure:

```markdown
# Title

## Section

Explanation with **bold** for key terms.

| Column | Column |
|--------|--------|
| data   | data   |

### Subsection

```csharp
// Code example with English comments and identifiers
public class Order { ... }
```

> Blockquote for tips and warnings

---

[← Previous: Title](prev-file.md) | [Next: Title →](next-file.md) | [Back to index](README.md)
```

### Navigation rules

- First file in a section: no "Previous" link
- Last file in a section: no "Next" link
- Always include `[Back to index](README.md)`
- Link text format: `← Previous: Topic Name` and `Next: Topic Name →`

## README Files

### Section README (`XX-section/README.md`)

```markdown
# XX - Section Title

## Contents

1. [Topic Name](01-file-name.md) - Brief description
2. [Topic Name](02-file-name.md) - Brief description

---

[Back to index](../README.md)
```

### Root README (`README.md`)

Lists all sections with a one-line description. Update it whenever adding a new section.

## Source Verification (MANDATORY)

**Every technical claim must be verified against official sources — before AND after being written.** Training-data recall is not acceptable for anything version-sensitive (API names, version numbers, release/EOL dates, default values, deprecations).

### Before adding or modifying content

1. Identify the primary official source for the topic and fetch it via `WebFetch`:

    | Area | Primary source |
    |---|---|
    | .NET / C# / ASP.NET Core / EF Core | `learn.microsoft.com` |
    | Angular | `angular.dev` (not legacy `angular.io`) |
    | RxJS | `rxjs.dev` |
    | AWS services | `docs.aws.amazon.com` + `aws.amazon.com/about-aws/whats-new` |
    | Azure services | `learn.microsoft.com/azure` |
    | MongoDB | `www.mongodb.com/docs` |
    | PostgreSQL / MySQL / Oracle | respective official docs |
    | Kafka / RabbitMQ / Redis | official project docs + release notes |
    | OpenAI | `platform.openai.com/docs` + `openai.com` announcements |
    | MCP / fast-moving SDKs | GitHub repo README + latest release notes |
    | HTTP / OAuth / OIDC / JWT | IETF RFCs (`datatracker.ietf.org`) |
    | Browser APIs / web security | MDN (`developer.mozilla.org`) |

2. Do NOT rely on training-data recall for:
    - API names, method signatures, attribute names
    - Version numbers, release dates, EOL/deprecation dates
    - Default values, flags, required parameters
    - Anything that changes between releases

3. For preview or fast-moving SDKs (MCP, HybridCache, Polly v8, Microsoft.Extensions.Http.Resilience, new Azure Functions models, etc.), always fetch the CURRENT README and release notes — preview APIs rename types between minor versions.

### After writing content

1. Re-verify every specific claim against the same official source. A claim's wording can drift from intent during writing.
2. Cross-check with community discussions for gotchas the docs don't mention:
    - GitHub Issues / Discussions on the official repo
    - Stack Overflow (prefer recent, high-voted answers)
    - Official blogs: `devblogs.microsoft.com`, `aws.amazon.com/blogs`, vendor engineering blogs (Confluent, Vercel, Jimmy Bogard, etc.)
3. If the official source does NOT directly back a claim, either:
    - Soften it ("observed behavior, not documented", "commonly cited"), OR
    - Drop it.

### What NOT to do

- NEVER guess an API shape from similar APIs — fetch the real doc.
- NEVER cite a version number or date without verifying it against release notes.
- NEVER keep content that "sounds right" if the official source contradicts it.
- NEVER trust a claim just because it was in the previous version of the file — verify on every edit.

### Why this matters

Training-data cutoffs drift. Preview SDKs rename types between minor versions. Blog posts go stale. A senior interviewer will call out any claim the official docs do not back, and one wrong claim destroys trust in the rest of the guide.

## When Adding New Content

1. **Verify against official docs** — see "Source Verification" above. This is the first step, always.
2. **Check if the topic already exists** — grep the repo before creating
3. **Choose the right section** — place it in the most relevant directory
4. **Number sequentially** — next number after the last file in that section
5. **Update the section README** — add the new file to the numbered list
6. **Update navigation links** — add "Next" to the previous file, "Previous" to the new file
7. **Update root README** — if the section description should change
8. **Cross-reference** — if the topic is mentioned in other files, link to it
9. **Add self-assessment questions** — whenever you add or modify a topic, add related questions to the corresponding file in `21-self-assessment/`. Use the collapsible `<details>` format with hidden answers and deep dive links
10. **Re-verify against official docs + community discussions** — see "Source Verification". Do this AFTER the content is written, before committing.

## When Modifying Content

- **Verify against official docs BEFORE and AFTER the edit** — see "Source Verification". An edit that "sounds right" but drifted from the current API is worse than no edit.
- Keep the same format and style as existing files
- Don't add Portuguese — everything in English
- Code examples should be practical and concise
- Use tables for comparisons (X vs Y)
- Use blockquotes (`>`) for tips and warnings
- Keep files focused — one topic per file, not everything in one giant file
- **Always add/update self-assessment questions** in `21-self-assessment/` when modifying topic content

## Reviewing a PR (MANDATORY)

When asked to **review a PR** (mine or any other), the review is worthless if it is not grounded. **Assume nothing. Verify everything.**

### Non-negotiable rules

- **NEVER approve a claim based on training-data recall.** Fetch the official source every time, even if the claim "sounds right" or "has been true for years".
- **Every factual claim in the diff must be individually verified** against its primary official source (see the table in "Source Verification"). "I already verified while writing" is not acceptable — re-verify on review.
- **Cross-check with community discussions** for gotchas the official docs don't mention (GitHub Issues, Stack Overflow top answers from the last 2 years, `devblogs.microsoft.com`, vendor engineering blogs).
- **When the official source does NOT directly back a claim**, report it. Do not soften silently — call it out so the user can decide to drop or hedge.
- **When the official source contradicts a claim**, report the contradiction with the exact quote and URL before editing.

### PR review checklist

1. **List every verifiable claim in the diff.** Version numbers, default values, menu names, keyboard shortcuts, method signatures, flag behaviors, operator names, benchmark numbers, availability ("since X version", "on by default in Y").
2. **Fetch the primary official source for each claim** via `WebFetch`. Do not reuse caches — each claim is its own verification.
3. **For each claim, record**: the exact quote from the official source, the URL, and whether it confirms, contradicts, or is silent on the claim.
4. **For claims the docs are silent on**, consult community sources (SO answers dated in the last 2 years, official blogs, issue threads). If still unverified, flag for removal or hedging.
5. **Produce a table** mapping each claim → verified (yes/no/partial) → action (keep / fix / drop / hedge).
6. **Only then apply corrections**, commit to the PR branch, and push.

### What counts as a verifiable claim

Anything that a reader could challenge with "where does it say that?":

- API / method / class / flag / attribute names
- Version numbers, release dates, EOL / deprecation dates
- Default values (enabled/disabled, thresholds, limits)
- Behavioral claims ("X triggers Y", "Z is not supported when W")
- Keyboard shortcuts, menu paths, UI element names
- Quoted text (must match the source verbatim)
- "Since version N" and "new in X" statements
- Comparisons between tools/versions (must be symmetrically verified)

### What NOT to do during a PR review

- Do not skip verification because the file was "just written" or "reviewed before".
- Do not verify only the "suspicious-looking" claims — verify every claim.
- Do not declare a review complete without the verification table.
- Do not mix verification with speculation — if a claim is unverified, label it so.

## Content Style

- **Interview-oriented**: write as if explaining in an interview, not a textbook
- **Practical**: include code examples, not just theory
- **Concise**: no filler. Lead with the answer
- **Opinionated**: include "when to use" and "when NOT to use"
- **Comparative**: use tables to compare alternatives (X vs Y)

## Current Sections

| # | Section | Directory |
|---|---------|-----------|
| 01 | C# Fundamentals | `01-csharp-fundamentals/` |
| 02 | Collections and LINQ | `02-collections-and-linq/` |
| 03 | Memory and Performance | `03-memory-and-performance/` |
| 04 | Concurrency and Parallelism | `04-concurrency-and-parallelism/` |
| 05 | Algorithms and Data Structures | `05-algorithms-and-data-structures/` |
| 06 | Architecture and Patterns | `06-architecture-and-patterns/` |
| 07 | HTTP and Web | `07-http-and-web/` |
| 08 | ASP.NET Core | `08-aspnet-core/` |
| 09 | Data Access | `09-data-access/` |
| 10 | Security | `10-security/` |
| 11 | Testing | `11-testing/` |
| 12 | DevOps | `12-devops/` |
| 13 | Cloud | `13-cloud/` |
| 14 | Observability | `14-observability/` |
| 15 | Reliability and SRE | `15-reliability-and-sre/` |
| 16 | Distributed Systems | `16-distributed-systems/` |
| 17 | System Design | `17-system-design/` |
| 18 | Angular | `18-angular/` |
| 19 | AI | `19-ai/` |
| 20 | Soft Skills | `20-soft-skills/` |
| 21 | Self-Assessment | `21-self-assessment/` |
