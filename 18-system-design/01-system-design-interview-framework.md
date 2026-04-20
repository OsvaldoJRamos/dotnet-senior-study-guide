# System Design Interview Framework

The hour is short and unstructured by design — the interviewer wants to see if you can impose the structure yourself. A senior candidate runs the session; a mid-level waits for prompts.

## The five phases

| Phase | Time (60-min) | Goal |
|---|---|---|
| 1. Clarify requirements | ~5 min | Functional, NFRs, out-of-scope |
| 2. Estimate | ~5 min | QPS, storage, bandwidth — just enough to inform the design |
| 3. High-level design | ~10 min | Boxes and arrows, API, data model |
| 4. Deep dive | ~25 min | Pick 1-2 components the interviewer cares about |
| 5. Trade-offs and wrap-up | ~10-15 min | Bottlenecks, failure modes, what you'd build next |

Keep a visible outline on the whiteboard. Don't skip phases — a strong deep-dive with no requirements discussion is a red flag.

## Phase 1 — Clarify requirements

**Always do functional + non-functional + out-of-scope.** Interviewers rarely volunteer NFRs; you have to pull them.

### Functional requirements (FRs)

Ask for the user-visible features. Write them as bullets on the board.

- "Users can post short text messages."
- "Users can follow other users."
- "Home feed shows posts from followed users, most recent first."

Push back on vagueness. *"What does 'recent' mean — last hour? Last day? Infinite scroll?"*

### Non-functional requirements (NFRs)

This is where seniors earn their stripes. Force numbers:

- **Scale** — how many users (DAU, MAU)? What's the read:write ratio?
- **Latency** — p50, p95, p99 budget for the critical path.
- **Availability** — three nines (8.76 h/yr down), four nines (52 min/yr), five nines (5 min/yr)?
- **Consistency** — strong, read-your-writes, eventual? What can be stale?
- **Durability** — is data loss acceptable? (Messages: no. View counts: yes.)
- **Geography** — single region, multi-region, global?
- **Security/privacy** — auth, PII, compliance (GDPR, HIPAA, PCI).

> If the interviewer says *"assume 100 million DAU"*, repeat it back and write it. You will use it in phase 2.

### Out-of-scope

Name what you are NOT designing so you don't get dragged in later. *"Spam detection, payments, and analytics are out of scope unless you want to add them back."*

## Phase 2 — Capacity estimation

Do the math out loud. The interviewer doesn't care if you nail the number — they care that you can reason about it. See [Capacity Estimation](02-capacity-estimation.md) for the toolkit.

Compute, at minimum:

- **QPS** (reads and writes separately — they scale differently).
- **Storage** (per year, plus replication factor).
- **Bandwidth** (ingress and egress).

Write each number on the board. You'll reference them when picking a database, cache size, or CDN.

## Phase 3 — High-level design

Start with a client → API gateway → service → database box diagram. Add cache, queue, and CDN only when justified by NFRs.

Cover three things:

1. **API surface** — 3-5 endpoints is enough. Request/response shape, not full OpenAPI.
2. **Data model** — core entities and their relationships. Hint at storage choice (SQL vs KV vs document vs graph).
3. **Data flow** — what happens on a write, what happens on a read.

> The diagram is a prop, not the deliverable. Keep it clean; move on quickly.

## Phase 4 — Deep dive

The interviewer will pick 1-2 components to probe. Common ones:

- **Hot component**: feed generation, ranking, search index.
- **Storage choice**: why Cassandra over Postgres? How do you shard?
- **Consistency**: how do you handle concurrent writes to the same row?
- **Failure mode**: what happens when the cache goes down? When a region is offline?

Be prepared to re-use the capacity numbers from phase 2. *"At 10 k writes/sec, a single Postgres primary is fine; at 100 k, we need sharding or Cassandra."*

## Phase 5 — Trade-offs and wrap-up

Close strong:

- **Bottlenecks** — what's the first thing that breaks at 10x scale?
- **Alternatives considered** — *"I chose fan-out on write; fan-out on read would be simpler but would hurt feed latency for power followers."* See [case study: news feed](08-case-study-news-feed.md).
- **What's next** — rate limiting, observability, disaster recovery, cost.

## Common mistakes

| Mistake | Why it's a red flag |
|---|---|
| Jumping to tech before requirements ("I'd use Kafka and Cassandra") | Cargo-culting; no evidence you picked them for a reason |
| Skipping NFRs | NFRs *are* the architecture; skipping them means you're guessing |
| Boiling the ocean — designing every component equally | Sub-linear progress; you'll run out of time before deep-diving anything |
| Ignoring the interviewer's steer | They know which rabbit hole is worth going down |
| Over-engineering day-1 | A feed for 10 k users doesn't need a Lambda architecture |
| Not using the numbers from phase 2 | If your capacity math never informs a decision, why did you do it? |
| Treating the whiteboard as a final deliverable | It's a communication tool; don't get precious about arrows |
| Silence during the deep dive | Narrate your thinking; the interviewer is grading reasoning, not final answers |

## Interview-style opener (script)

*"Before I design anything, let me align on what we're building. Who's the user, and what are the top 2-3 things they do? Then I want to pin down scale — rough DAU and read/write ratio — and any latency or availability targets you care about. I'll note anything I assume so you can correct me. Then I'll do a quick capacity estimate, sketch the high-level, and we'll deep-dive wherever you want."*

That single paragraph telegraphs: *I run the hour, I value requirements, I'll check my assumptions.* It's the strongest signal you can send in the first 30 seconds.

---

[Next: Capacity Estimation →](02-capacity-estimation.md) | [Back to index](README.md)
