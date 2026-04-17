# Distributed Systems

> Read the questions, think about your answer, then click to reveal.

---

### 1. State the CAP theorem formally. What's wrong with the "pick 2 of 3" framing?

<details>
<summary>Reveal answer</summary>

Gilbert & Lynch (2002) proved: no distributed data store can **simultaneously** provide Consistency (linearizability — every read sees the latest write or an error), Availability (every non-failing node responds successfully), and Partition tolerance (the system keeps working despite arbitrary message loss).

"Pick 2 of 3" is misleading because **partitions aren't a choice** — they happen. Any real distributed system has to tolerate them. So the real trade-off is only visible **during a partition**: do you stay consistent (refuse minority-side requests → CP) or stay available (serve both sides, reconcile later → AP)? During normal operation, a well-designed system is both C and A.

A better framing is Abadi's **PACELC** (2012): if partition, choose A or C; **else**, choose latency or consistency. That captures the cost you pay every request, not only when the network fails.

Deep dive: [CAP Theorem](../16-distributed-systems/01-cap-theorem.md)

</details>

---

### 2. Explain the difference between strong, causal, and eventual consistency. Give a business case for each.

<details>
<summary>Reveal answer</summary>

- **Strong / linearizable**: every read anywhere sees the latest write. *Money transfers, inventory, distributed locks, leader election.* Anything where "stale" equals "wrong."
- **Causal**: operations that are causally related are seen in the same order by everyone; concurrent ones may differ. *Social feeds — a reply must never appear before its parent post, but two unrelated posts can reorder harmlessly.*
- **Eventual**: replicas converge "eventually" with no ordering guarantees. *Analytics counters, product catalogues, search indexes, content caches — data you can tolerate being seconds out of date.*

Cosmos DB exposes these (and two in-between — Session and Bounded Staleness) as explicit account-level choices. The default `Session` level gives read-your-writes + monotonic reads within one client session, which is the sweet spot for most web apps — strong-feeling UX at eventual-consistency cost.

Deep dive: [Consistency Models](../16-distributed-systems/02-consistency-models.md)

</details>

---

### 3. Why is idempotency mandatory in distributed systems? What delivery semantic does Kafka provide out of the box?

<details>
<summary>Reveal answer</summary>

The Two Generals Problem: on an unreliable network, a client that times out **cannot know** whether its request succeeded. The only safe response is to retry — and the only way retries are safe is if the server operation is idempotent.

Kafka's default delivery is **at-least-once**. Producers retry on failure; consumers can see the same message twice after a rebalance or crash. "Exactly-once" in production means **at-least-once delivery + idempotent consumer = effectively-once processing**. Kafka's EOS feature (`enable.idempotence=true`, transactional API, `processing.guarantee=exactly_once` in Streams) provides true exactly-once **within Kafka** (topic-to-topic). Per Confluent's own docs: if the app makes an RPC to an external store, the side effect there is NOT covered by EOS — you still need idempotent consumers on the destination.

Deep dive: [Idempotency](../16-distributed-systems/03-idempotency.md)

</details>

---

### 4. The IETF Idempotency-Key draft specifies which HTTP status codes for "same key, different body" and "same key, request in flight"?

<details>
<summary>Reveal answer</summary>

- **Same key, different request body** → `HTTP 422 Unprocessable Content` ("If there is an attempt to reuse an idempotency key with a different request payload").
- **Same key, first request still in flight** → `HTTP 409 Conflict` ("The request was retried before the original request completed").
- **Missing key on an endpoint that requires one** → `HTTP 400 Bad Request`.
- The header applies to `POST` and `PATCH` — the draft explicitly notes `GET`/`HEAD`/`PUT`/`DELETE`/`OPTIONS` are already idempotent per RFC 9110.

The cache is an optimisation; the database `UNIQUE` constraint on the key is the real guarantee. Reject-with-422 and serialize-with-409 are only correct if the store reflects the winner of the race.

Deep dive: [Idempotency](../16-distributed-systems/03-idempotency.md)

</details>

---

### 5. What is the dual-write problem and how does the Outbox pattern solve it?

<details>
<summary>Reveal answer</summary>

A service needs to atomically (a) update its database and (b) publish a message to a broker. These are different systems, so no single transaction spans both. Every crash-ordering is a bug: commit-then-crash loses the message; publish-then-crash fires events for state that never persisted. Chris Richardson: *"it is not viable to use a traditional distributed transaction (2PC) that spans the database and the message broker."*

**Outbox**: in the same DB transaction that writes the domain change, insert a row into an `outbox` table in the same database. A separate relay (polling publisher or CDC tailing via Debezium) reads outbox rows and publishes them to the broker, marking rows as sent on acknowledgement.

Guarantees: never publish for a rolled-back change; never lose a message once the change committed; at-least-once delivery (so consumers must dedupe on message ID — the **inbox** pattern closes the loop). MassTransit ships both modes (`InMemoryOutbox` and EF-backed transactional outbox); NServiceBus and Wolverine have equivalents.

Deep dive: [Outbox Pattern](../16-distributed-systems/04-outbox-pattern.md)

</details>

---

### 6. Choreography vs orchestration sagas — when would you pick each? What's the hardest part of either?

<details>
<summary>Reveal answer</summary>

**Choreography**: services publish events and react to events from others; no central coordinator. Good for short flows (2–3 steps) with obvious participant roles. Cons: flow is implicit, cycles and races are easy to introduce, debugging needs distributed tracing.

**Orchestration**: a central state machine issues commands and reacts to replies. Good for longer flows (4+ steps), branching, and error paths because the whole saga is readable in one place. Cons: orchestrator is a form of coupling and a single point of failure (though it should be stateful and resumable).

**Hardest part either way**: compensating transactions. They're business-semantic reversals, not DB rollbacks — `RefundPayment`, `ReleaseStock`. They must be idempotent (messages redeliver) and must eventually succeed or escalate. Some actions — "send email", "ship a physical item" — are uncompensable, so you sequence them last or design them to be cancelable within a window. And sagas lack isolation: concurrent sagas can see each other's partial state, so you need countermeasures (semantic locks, commutative updates, pessimistic view, etc. per Richardson).

Deep dive: [Saga Pattern](../16-distributed-systems/05-saga-pattern.md)

</details>

---

### 7. Why is Two-Phase Commit avoided in microservices? When does it still make sense?

<details>
<summary>Reveal answer</summary>

2PC is a **blocking** protocol. After voting `YES` in phase 1, a participant holds its locks until the coordinator sends commit or abort. If the coordinator dies between phases, participants are stuck — they can't safely commit (maybe others voted no) and can't abort (maybe others already committed). Practical pain: locks held for remote latency kill throughput; the coordinator becomes a single point of failure; and most microservice participants (Kafka, REST services, SaaS APIs) don't speak XA, so 2PC isn't even an option.

The modern recipe is **one ACID transaction per service + outbox + saga + idempotent consumers**. You trade cross-step isolation for availability and scale.

2PC still fits: two XA-capable databases on a LAN, short-lived low-concurrency operations, legacy DTC-on-Windows systems. It does **not** fit cross-service microservice workflows over a WAN.

Deep dive: [Distributed Transactions](../16-distributed-systems/06-distributed-transactions.md)

</details>

---

### 8. Walk me through Raft leader election. How many nodes tolerate how many failures, and why?

<details>
<summary>Reveal answer</summary>

Every follower runs a randomized election timeout (Raft paper suggests 150–300 ms). If no heartbeat arrives within the timeout, the follower becomes a **candidate**, increments the term, votes for itself, and sends `RequestVote` RPCs to all other servers. A candidate wins if a **majority** of the cluster grants its vote. On a split vote, another randomised timeout triggers a fresh election. The winner starts sending heartbeats immediately to suppress new elections.

For a cluster of *N* nodes, majority = `floor(N/2) + 1`, and failure tolerance = `N - majority`:

| *N* | Majority | Failures tolerated |
|---|---|---|
| 3 | 2 | 1 |
| 5 | 3 | 2 |
| 7 | 4 | 3 |

**Always use odd sizes.** A 4-node cluster tolerates the same 1 failure as 3 nodes, at higher cost and with doubled split-vote probability. And Raft is **crash-stop tolerant, not Byzantine** — it assumes honest participants.

Deep dive: [Consensus — Raft and Paxos](../16-distributed-systems/07-consensus-raft-paxos.md)

</details>

---

### 9. Name three systems built on Raft and say what they use it for.

<details>
<summary>Reveal answer</summary>

- **etcd** — distributed KV store backing Kubernetes cluster state. Raft gives it linearizable reads/writes on the control plane.
- **Consul** — HashiCorp's service discovery / mesh. Raft runs the server cluster for the KV store and service catalog; gossip (SWIM) handles larger-scale membership.
- **CockroachDB / TiKV** — per-range replication. Each "range" is its own Raft group of 3 or 5 replicas; cross-range transactions coordinate atop.
- **Kafka KRaft** — Kafka's move away from ZooKeeper (KIP-500) replaces the controller with a Raft-based metadata quorum. Marked production-ready for new clusters in 3.3 (KIP-833); ZooKeeper fully removed in 4.0.
- **MongoDB replica sets** — primary election and log replication use a Raft-inspired protocol (protocol version 1).

Interview signal: 99% of the time, a senior should **use one of these**, not implement their own Raft. Running a correct Raft library in production is multi-year work.

Deep dive: [Consensus — Raft and Paxos](../16-distributed-systems/07-consensus-raft-paxos.md)

</details>

---

### 10. Explain Lamport timestamps. What guarantee do they give, and what guarantee do they NOT give?

<details>
<summary>Reveal answer</summary>

Each process keeps an integer counter `C`. Lamport's (1978) rules:

1. Increment `C` before each local event.
2. Attach `C` to every outgoing message.
3. On receive of a message with timestamp *t*: `C := max(C, t) + 1`, then process.

**Guarantee (clock consistency):** if event *a* happened-before event *b*, then `C(a) < C(b)`.

**NOT a guarantee:** `C(a) < C(b)` does NOT imply *a* → *b*. Two concurrent events can have different Lamport values. So Lamport timestamps detect "certainly after" but cannot distinguish "happened-before" from "concurrent". If you need that — for instance, to detect conflicts in a multi-master replicated store — you need **vector clocks** (Fidge 1988 / Mattern 1988), where `V(a) < V(b)` iff *a* → *b*. Vector clocks cost O(N) space per timestamp, so they don't scale past small replica sets.

Deep dive: [Clocks and Ordering](../16-distributed-systems/08-clocks-and-ordering.md)

</details>

---

### 11. Why shouldn't you use `DateTime.UtcNow` to measure elapsed time or to order events across services?

<details>
<summary>Reveal answer</summary>

`DateTime.UtcNow` reads the OS **wall clock**, which can:

- Jump backwards (NTP correction, leap seconds, admin adjustment).
- Drift tens of ms from other machines even with NTP.
- Pause during a GC or VM live-migration.

For **elapsed time within one process**: use a monotonic clock — `Stopwatch` in .NET, `CLOCK_MONOTONIC` on Linux. `Stopwatch.Elapsed` is monotonic and safe to subtract.

For **ordering events across services**: don't trust wall-clock timestamps for causal ordering. Use a logical clock (Lamport or Hybrid Logical Clock) in the message payload, or serialize through a single partition (Kafka partition key) where the broker's own ordering applies.

Wall-clock timestamps are fine for human-readable logs and audit records — just don't build correctness on them.

Deep dive: [Clocks and Ordering](../16-distributed-systems/08-clocks-and-ordering.md)

</details>

---

### 12. "We need exactly-once across our payment service, Kafka, and our fraud-check service." How do you actually deliver this?

<details>
<summary>Reveal answer</summary>

Exactly-once **delivery** across a network is impossible (Two Generals). The production recipe is **at-least-once delivery + idempotent processing end-to-end = effectively-once**.

Walk-through:

1. **Payment service → Kafka**: use the [Outbox pattern](../16-distributed-systems/04-outbox-pattern.md). Single DB transaction writes the charge + an outbox row; a relay publishes the outbox row to Kafka with at-least-once semantics.
2. **Inside Kafka (fraud-check reads, processes, writes to an output topic)**: enable Kafka EOS — `enable.idempotence=true` on producers, transactional producer with `transactional.id`, consumers reading from the output topic use `isolation.level=read_committed`, Kafka Streams `processing.guarantee=exactly_once`. This makes the read-process-write loop atomic **within Kafka**.
3. **Fraud-check writes to its own DB**: EOS does NOT cover this step (Confluent is explicit about this). Fraud-check must be idempotent — dedupe on Kafka message ID via an inbox table with a unique constraint, and use upserts / optimistic concurrency for domain state.
4. **Saga layer** for business invariants spanning services: orchestrate with compensations for failures past acknowledgement.
5. **Client-facing retries** (API gateway, public `POST`): require `Idempotency-Key` header backed by a DB unique constraint.

The honest answer in an interview: there is no magic "exactly-once" switch. You compose idempotent layers until duplicates become invisible.

Deep dive: [Distributed Transactions](../16-distributed-systems/06-distributed-transactions.md) and [Idempotency](../16-distributed-systems/03-idempotency.md)

</details>

---

[Back to index](README.md)
