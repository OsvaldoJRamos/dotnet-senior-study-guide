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

## When Adding New Content

1. **Check if the topic already exists** — grep the repo before creating
2. **Choose the right section** — place it in the most relevant directory
3. **Number sequentially** — next number after the last file in that section
4. **Update the section README** — add the new file to the numbered list
5. **Update navigation links** — add "Next" to the previous file, "Previous" to the new file
6. **Update root README** — if the section description should change
7. **Cross-reference** — if the topic is mentioned in other files, link to it
8. **Add self-assessment questions** — whenever you add or modify a topic, add related questions to the corresponding file in `15-self-assessment/`. Use the collapsible `<details>` format with hidden answers and deep dive links

## When Modifying Content

- Keep the same format and style as existing files
- Don't add Portuguese — everything in English
- Code examples should be practical and concise
- Use tables for comparisons (X vs Y)
- Use blockquotes (`>`) for tips and warnings
- Keep files focused — one topic per file, not everything in one giant file
- **Always add/update self-assessment questions** in `15-self-assessment/` when modifying topic content

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
| 10 | Testing | `10-testing/` |
| 11 | DevOps | `11-devops/` |
| 12 | Cloud | `12-cloud/` |
| 13 | Angular | `13-angular/` |
| 14 | AI | `14-ai/` |
| 15 | Self-Assessment | `15-self-assessment/` |
