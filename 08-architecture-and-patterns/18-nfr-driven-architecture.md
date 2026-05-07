# Non-Functional Requirements Drive the Architecture

Functional requirements describe *what* the system does — authenticate users, issue refunds, calculate tax. They determine the **code**. Non-functional requirements (NFRs) describe *how well* the system has to do it — at what latency, at what scale, with what uptime, under what regulations. They determine the **architecture**.

If you only collect functional requirements, you'll build the right feature on the wrong architecture — and discover it the first time traffic triples or a disk fails.

## The three classes of requirement

| Kind | Answers | Example |
|---|---|---|
| **Business** | *Why* are we doing this? | "Reduce cart-abandonment by 10% this quarter." |
| **Functional** | *What* must the system do? | "The user can apply a promo code at checkout." |
| **Non-Functional (NFR)** | *How* must it behave? | "Checkout p95 latency ≤ 300 ms at 1,000 rps." |

A business requirement decomposes into one or more functional requirements **plus** the NFRs that constrain them. All three must be aligned — a functional story with no NFR is under-specified.

## The canonical NFR categories

ISO/IEC 25010 — the international standard for software product quality — groups the quality characteristics into the set below. You don't have to memorize the standard, but the names show up in RFPs and interview questions. (`iso25000.com/iso-25000-standards/iso-25010`)

| Characteristic | Typical metrics | Architectural levers |
|---|---|---|
| **Functional Suitability** | Completeness, correctness | Unit/integration tests, acceptance criteria |
| **Performance Efficiency** | p50/p95/p99 latency, throughput (rps), CPU/memory footprint | Caching, async, efficient queries, CDN, indexing |
| **Compatibility** | Protocols supported, co-existence with other systems | Versioned APIs, adapters, contract tests |
| **Interaction Capability** (a.k.a. Usability) | Task success rate, time-on-task | UX design, accessibility, BFF per client |
| **Reliability** | Availability (%), MTBF, MTTR, RPO, RPO | Redundancy, health checks, retries, backups, DR drills |
| **Security** | CIA (confidentiality/integrity/availability), authn/authz, audit | AuthN/Z at gateway, encryption in transit + at rest, secret management, WAF |
| **Maintainability** | Cycle time, defect density, onboarding time | Clean Architecture, tests, ADRs, modular boundaries |
| **Portability** | Ability to move to another environment | Container images, infra-as-code, no cloud-specific APIs in core |

## How NFRs translate into architecture decisions

This is the senior signal: given an NFR, pick the minimum architecture that satisfies it. Given a proposed architecture, predict what NFR it breaks.

### Latency: "p95 ≤ 200 ms"

| Architectural response | Trade-off |
|---|---|
| Cache read-heavy paths (Redis, CDN, in-memory) | Invalidation complexity; stale reads |
| Move aggregation off the hot path (CQRS read model, BFF) | Extra system to build and operate |
| Avoid synchronous fan-out; pre-compute | Eventual consistency in reads |
| Use async IO end-to-end (async/await, no blocking) | Steeper debugging of cross-service flows |
| Co-locate services and DB (same region / AZ) | Harder DR story |

### Throughput / scale: "10k rps, must scale with traffic"

| Response | Trade-off |
|---|---|
| Stateless compute + horizontal scaling | Need external state (Redis, DB, blob) |
| Partition / shard data | Rebalancing pain; cross-shard queries |
| Async processing behind a queue | Can't synchronously return the final result |
| Read replicas | Replica lag; write-after-read anomalies |
| Bulk / batch APIs | UX has to accept coarser operations |

### Availability: "99.95% uptime"

99.95% = ~4.4 hours/year of allowed downtime. The number alone constrains:

| Response | Trade-off |
|---|---|
| Multi-AZ deployment | 2× infra cost |
| Multi-region active–active | Large jump in complexity (conflict resolution, data residency) |
| Load balancer + health checks + auto-rollback | You need observability good enough to trust the health signal |
| Graceful degradation (serve cached data when backend fails) | UX must tolerate it |
| Chaos engineering / game days | Cultural investment, not just tooling |

> Availability targets must be stated with a window. "99.95% availability" is meaningless without "measured monthly, excluding planned maintenance of up to 2 hours/month".

### Consistency: "I can't read what I just wrote" vs "My balance is never wrong"

This is where CAP starts to bite. You can't have *strong consistency*, *high availability*, and *partition tolerance* simultaneously — so pick two, or split the system so different parts pick differently.

| NFR | Architectural response |
|---|---|
| Strong consistency on money movement | Single primary DB, synchronous transaction, read-your-writes via the write side |
| Read-after-write on user profile edits | Session-sticky routing to the write replica, or short read-your-writes window |
| Eventual consistency acceptable (dashboards, search) | Projections from an event stream, async replication |

### PACELC extension

**P**artition: choose **A**vailability or **C**onsistency. **E**lse (no partition): choose **L**atency or **C**onsistency. Every datastore has a PACELC profile — Cassandra is AP/EL, traditional SQL Server is CP/EC, and so on. Know your store's profile before promising strong consistency.

### Security: "GDPR / LGPD / SOC 2 / HIPAA"

Regulatory NFRs drive structural decisions:

| Requirement | Response |
|---|---|
| Data residency (EU, Brazil, US-only) | Region-scoped deployments; no implicit cross-region replication |
| Right to be forgotten | No append-only logs with PII in the clear; crypto-shredding keys per subject |
| Audit trail | Event store or immutable append-only log + retention policy |
| Least privilege | Scoped credentials per service; secrets in a vault, not config files |
| Encryption at rest + in transit | Managed keys (Azure Key Vault / AWS KMS); TLS enforced end to end |

### Maintainability: "team must ship features weekly"

Not every NFR shows up in an SLO. The ones about **team throughput** are just as architectural:

- Bounded contexts aligned to team ownership (see [DDD](13-ddd.md)).
- CI pipeline that gives a green build in under 15 minutes.
- Observability good enough that no deployment is a gamble (see [C4 and ADRs](17-design-docs-c4-adr.md)).

## The senior workflow

1. **Elicit NFRs explicitly**, with numbers. "Fast" isn't an NFR; "p95 ≤ 300 ms at 500 rps" is.
2. **Pick the binding NFR per feature** — usually latency, availability, or consistency will be the one that forces an architectural decision. The others fall out.
3. **Map NFRs to architecture options**, with trade-offs. Every option lists its cost.
4. **Write an ADR** for each significant decision (see [Design Docs, C4 and ADRs](17-design-docs-c4-adr.md)).
5. **Measure.** A claimed NFR that isn't monitored is a claim, not a guarantee. Every SLO needs a dashboard and an alert.

## Pitfalls

- **Vague NFRs.** "Must be fast" → impossible to design for, impossible to verify. Force a number and a percentile.
- **NFRs without a measurement plan.** "99.9%" with no uptime dashboard is wishful thinking.
- **Treating all NFRs as equal.** Usually one or two dominate (latency for a checkout page; availability for a payment API; consistency for a ledger). Design for the binding ones; the rest are constraints to satisfy, not to optimize.
- **Retrofitting NFRs.** Adding "must be HIPAA-compliant" after the design decisions are made usually costs more than a rewrite of the affected modules.
- **Picking an architecture, then rationalizing the NFRs.** "We need microservices" is a solution looking for a problem. Pick the architecture that the NFRs force, and no more.

## Rule of thumb

> Every significant architectural decision should point to the NFR it satisfies. If you can't name the NFR, you're adding complexity for free.

---

[← Previous: Design Docs, C4 and ADRs](17-design-docs-c4-adr.md) | [Back to index](README.md) | [Next: UML Diagrams →](19-uml-diagrams.md)
