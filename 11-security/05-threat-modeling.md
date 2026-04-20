# Threat Modeling

Threat modeling is *thinking about how your system can be attacked, **before** it's attacked*. Done well, it finds design-level flaws that no library or scanner can catch (OWASP A04:2021 — Insecure Design). Done badly, it's a two-hour meeting that produces a PDF nobody reads.

## The four questions (Shostack)

Adam Shostack's framing — the one the Microsoft SDL leans on — reduces threat modeling to four questions, asked in order:

1. **What are we building?** Describe the system. Usually a Data Flow Diagram (DFD).
2. **What can go wrong?** Enumerate threats — STRIDE gives you categories.
3. **What are we going to do about it?** Mitigations, accepted risks, or transfers.
4. **Did we do a good job?** Review the model against the shipped system; update it.

If a threat model doesn't answer all four, it isn't one.

## STRIDE

Microsoft's threat taxonomy. Each letter maps to one security property it attacks.

| Letter | Threat | Property violated | Typical example |
|---|---|---|---|
| **S** | Spoofing | Authentication | Stolen password, forged session cookie, JWT with `alg=none` |
| **T** | Tampering | Integrity | Modifying data in transit, SQL injection, signed URL forgery |
| **R** | Repudiation | Non-repudiation | User denies an action and there's no audit trail to prove it |
| **I** | Information disclosure | Confidentiality | Verbose error page leaking stack trace, cloud metadata endpoint exposure |
| **D** | Denial of service | Availability | Unbounded query, unthrottled login endpoint, ReDoS |
| **E** | Elevation of privilege | Authorization | IDOR, missing server-side check, broken access control |

The Microsoft Threat Modeling Tool supports **STRIDE per element** — each DFD element (process, data store, data flow, external interactor) gets its applicable subset of STRIDE threats auto-generated.

## Data Flow Diagrams (DFDs)

The canonical threat-modeling diagram. Keep the notation tight:

| Shape | Meaning |
|---|---|
| Circle | Process (code you run) |
| Two parallel lines | Data store (DB, cache, queue, file) |
| Arrow | Data flow (every arrow is labeled with what flows) |
| Rectangle | External entity (user, third party) |
| Dashed line | **Trust boundary** — the most important element |

The value of a DFD is not in being pretty; it's in **marking the trust boundaries**. Every arrow crossing a trust boundary is where attackers live. Walk each crossing and apply STRIDE.

### Minimal example

```
[User] --HTTPS--> (Web App) --TLS--> (Auth Service) --TLS--> [[Identity Store]]
                   |
                   +--TLS--> [[Orders DB]]
```

Trust boundaries:

- Browser ↔ Web App (internet boundary) — apply S, T, I, D.
- Web App ↔ Auth Service — likely internal network; still enforce mTLS and apply S, T.
- Web App ↔ Orders DB — apply T, I, E (SQL injection, credential compromise, unauthorized read).

## Attack trees

Complementary to STRIDE. Put a **goal** at the root, then decompose into the **methods** to reach it.

```
Goal: steal a user's auth cookie
├── XSS in the app
│   ├── Reflected XSS in a search endpoint
│   └── Stored XSS in a profile field
├── Intercept on the network
│   ├── Downgrade to HTTP (no HSTS)
│   └── Malicious Wi-Fi + user ignores cert warning
└── Physical access
    ├── Unlocked laptop
    └── Cookie sync to attacker's device
```

Attack trees are particularly useful for **non-obvious goals** — supply-chain compromise, insider threat, regulatory data exfiltration. STRIDE is better for breadth; attack trees are better for depth on a specific concern.

## Microsoft Threat Modeling Tool

From the Microsoft docs, the Threat Modeling Tool *"is a core element of the Microsoft Security Development Lifecycle (SDL)"*. It lets you draw a DFD and automatically enumerate STRIDE threats per element, attach mitigations, and export a report.

Strengths:

- Explicit **trust boundaries** and per-element STRIDE.
- Templates for Azure components.
- Free, Windows-only GUI (at time of writing).

Limitations:

- Windows-only desktop tool.
- The auto-generated threat list is generic — still needs human pruning.
- Templates for AWS and multi-cloud are limited; community templates fill some gaps.

Alternatives interviewers recognize:

- **OWASP Threat Dragon** — cross-platform (Electron), STRIDE + generic templates, free.
- **IriusRisk**, **Tutamantic**, **SecuriCAD** — commercial, integrate with Jira/CI.
- **`threatgrammar` / pytm** — threat modeling as code; fits a GitOps workflow.

## When to run a threat-modeling session

Do it often enough that it stays cheap:

- **Before a new service is built.** Cheapest time to change the design.
- **Before a trust boundary changes.** Going from "internal API" to "partner API", adding a third party, multi-tenanting a previously single-tenant system.
- **After a security incident or a near miss.** Use what you learned.
- **Annually, for every tier-1 system**, as a regular review.

Don't try to threat-model *everything*. A trivial CRUD internal tool probably doesn't deserve an hour — but the billing pipeline does.

## Running the session

A workable 90-minute format:

1. **10 min** — Walk the DFD. If there isn't one, draw it during the session.
2. **15 min** — Identify trust boundaries and data classifications (PII, PCI, PHI, internal only).
3. **45 min** — STRIDE walk on each element / each boundary-crossing arrow. Capture in a shared spreadsheet: *Element · Threat · Existing control · Gap · Owner*.
4. **15 min** — Prioritize. Use DREAD (Damage, Reproducibility, Exploitability, Affected users, Discoverability) or a simple High/Medium/Low — just be consistent.
5. **5 min** — Assign owners and dates; add gaps as tickets with a security label.

> The output that matters is the **tickets**, not the diagram. A beautiful Threat Modeling Tool file with no follow-up is waste.

## Prioritization approaches

No single model is universal; interviewers expect you to name at least two:

- **DREAD** (Microsoft, historical) — sum of 1–10 scores across five dimensions. Often flagged for subjectivity; still useful as a conversation prompt.
- **CVSS** (FIRST) — 0–10 base score; industry standard for CVEs.
- **OWASP Risk Rating** — Likelihood × Impact on 0–9 scale with factor guidance.
- **Simple risk matrix** — Likelihood {Low, Med, High} × Impact {Low, Med, High}. Often the right tool for a project meeting.

## Anti-patterns

- **Threat model once, then never touch it.** The system changed; the model hasn't.
- **Security team does the threat model alone.** Product and engineering must be in the room — they make the trade-offs.
- **Confuse threat modeling with a checklist.** OWASP ASVS is a *verification* checklist. Threat modeling is about **design-specific** risks the checklist can't anticipate.
- **Skipping mitigations for "low likelihood".** Power-law distributions: the one-in-ten-thousand event is often the one that ends the company.
- **No trust boundaries on the DFD.** Without them the model is just an architecture diagram with extra arrows.

## Rule of thumb

A threat model is a success when a new engineer can read it in 15 minutes, point at a trust boundary, and name what's protecting it. If the answer is *"uh, HTTPS I guess"* on a boundary that touches customer data, the session did its job.

---

[← Previous: Secrets Management](04-secrets-management.md) | [Next: Supply Chain Security →](06-supply-chain-security.md) | [Back to index](README.md)
