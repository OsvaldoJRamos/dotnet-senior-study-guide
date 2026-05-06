# Distributed Transactions

## The goal

A business operation spans multiple resources (databases, brokers, services). You want the result to be atomic — **all commit, or none commit**. In a single database, ACID gives you this for free. Across resources, you do not.

## Two-Phase Commit (2PC)

The classical answer. A **coordinator** runs two rounds with all participants:

### Phase 1 — Prepare (voting)

Coordinator asks every participant: *"can you commit?"* Each participant executes its work, takes locks, writes to its undo/redo log, and replies `YES` (ready to commit, cannot back out) or `NO` (must abort).

### Phase 2 — Commit / Abort

If every participant voted `YES`, the coordinator sends `COMMIT` to all. Otherwise it sends `ABORT`. Participants finalize and release locks.

```text
         ┌──── Phase 1: PREPARE ────┐
Coord ──▶│                           │◀── YES / NO
Coord ──▶│   Participants lock &     │
         │   write prepare record    │
         └───────────────────────────┘

         ┌──── Phase 2: COMMIT / ABORT ────┐
Coord ──▶│                                  │◀── ACK
         │   Participants finalize         │
         └──────────────────────────────────┘
```

### Why 2PC is avoided in microservices

**Blocking.** After voting `YES`, a participant holds its locks until the coordinator tells it to commit or abort. If the coordinator dies after collecting votes but before sending the decision, participants are stuck — they cannot safely roll back (maybe the others already committed) and cannot commit (maybe the others aborted). This is the protocol's defining flaw: the standard database-textbook treatment (Bernstein, Hadzilacos & Goodman, *Concurrency Control and Recovery in Database Systems*, 1987) characterises 2PC as a **blocking protocol** — once a participant sends `PREPARED`, "until it receives a message containing the coordinator's decision, it is unable to commit or abort."

**Practical consequences:**

- **Locks held for remote latency.** A 50 ms cross-service prepare holds row locks in every participant for 50 ms. Throughput collapses under contention.
- **Coordinator availability requirement.** The coordinator becomes a single point of failure whose availability caps the whole system's availability.
- **Heterogeneity problem.** 2PC needs XA-compliant resource managers. Kafka, HTTP services, most REST APIs, and many SaaS systems do not speak XA. In a microservices world, most participants aren't eligible.
- **Partition + coordinator failure.** Unrecoverable without human intervention.

For these reasons, the microservices community — Richardson, Fowler, the DDD crowd — treats 2PC as a **non-option** for cross-service business operations. Single-DB ACID plus [sagas](05-saga-pattern.md) plus [outbox](04-outbox-pattern.md) is the modern recipe.

### Where 2PC still makes sense

- **Classic enterprise integration** with two XA-capable databases on a LAN (e.g., Oracle + Oracle, SQL Server + MSMQ in a DTC transaction).
- **Short-lived, low-concurrency** operations where lock-hold time is acceptable.
- **Not** cross-service microservice workflows over a WAN.

### .NET and DTC

`System.Transactions.TransactionScope` and MS DTC (Microsoft Distributed Transaction Coordinator) can still enlist multiple resources in a 2PC transaction on Windows. This is legacy territory — most greenfield .NET systems avoid it. The .NET docs on `TransactionScope` document the behaviour for anyone maintaining such systems.

## The modern alternatives

Instead of a distributed transaction, you compose **local transactions** with messaging:

| Pattern | What it gives you | Where it fits |
|---|---|---|
| [Outbox Pattern](04-outbox-pattern.md) | Atomic "change local state + emit message" | Every service that publishes events after a DB change |
| [Saga Pattern](05-saga-pattern.md) | Coordination of a multi-step workflow with compensations | Business operations that span services |
| Idempotent consumers | Tolerates at-least-once delivery | Every message consumer |
| [Consensus](07-consensus-raft-paxos.md) (Raft/Paxos) | Single decision agreed by a replica set | Within a system's control plane, not across services |

The pattern is: **one local ACID transaction per step**, messages to coordinate, compensations to undo. You give up cross-step isolation and get availability + scale in return.

## Kafka exactly-once semantics (EOS)

Kafka is the one place where a "distributed transaction" is practical in a messaging system — but **only within Kafka**.

### Idempotent producer

With `enable.idempotence=true`, the producer attaches a sequence number to each batch. Brokers dedupe retries: *"each batch of messages sent to Kafka will contain a sequence number that the broker will use to dedupe any duplicate send."* (Confluent)

Result: no duplicate messages from producer retries for a single partition.

### Transactional producer

Producers configure `transactional.id` and use `initTransactions` / `beginTransaction` / `sendOffsetsToTransaction` / `commitTransaction`. Writes to multiple partitions are either **all visible or all aborted**.

Consumers set `isolation.level=read_committed` to skip uncommitted and aborted messages.

### Kafka Streams `processing.guarantee=exactly_once`

For the read-process-write pattern inside Streams, EOS covers the entire cycle: consume from input topic, process, write to output topic, and commit offsets — all atomically. This is the "exactly-once" that Confluent markets.

### The critical caveat

Confluent states explicitly: *"exactly-once semantics is guaranteed within the scope of Kafka Streams' internal processing only; for example, if the event streaming app...makes an RPC call to update some remote stores...the resulting side effects would not be guaranteed exactly once."*

So:

- **Inside Kafka (topic → topic)**: EOS works.
- **Kafka → external DB / service**: NOT covered by EOS. You still need idempotent consumers on the destination side (see [Outbox Pattern](04-outbox-pattern.md) when writing from the service back to Kafka).

### Performance

Confluent reports roughly a **3% throughput overhead** for idempotent/transactional producers with 1 KB messages and 100 ms transaction commits; 15–30% degradation for Kafka Streams at a 100 ms commit interval, shrinking as the commit interval grows.

## Decision rule for "I need atomicity across services"

```text
Is the operation truly cross-service (different DBs, different teams)?
├── No — one DB ──▶ ACID transaction. Done.
└── Yes
    ├── Can it be modeled as "emit an event, react to it"?
    │   ├── Yes ──▶ Outbox pattern on the producer + idempotent consumer.
    │   └── No — need multi-step workflow
    │       └──▶ Saga (choreography or orchestration) + per-step outbox + idempotent consumers.
    └── Am I tempted to reach for 2PC?
        └──▶ Reconsider. The only reasonable cases are two XA databases on a LAN, and even those are slowly being rewritten.
```

## Pitfalls

- **"We'll add 2PC later if we need it."** You won't. By the time you need it, the system has grown beyond 2PC's reach. Design for sagas from day one.
- **Treating Kafka EOS as cross-system exactly-once.** It isn't. Every sink outside Kafka is still at-least-once for your code.
- **Forgetting consumer idempotency.** Producer-side EOS (dedup by sequence number) only protects the topic. Consumers on downstream systems still see duplicates during rebalances and restarts.
- **Mixing TransactionScope with async code on Windows.** Ambient transactions and `await` do not play well by default. Use `TransactionScopeAsyncFlowOption.Enabled` explicitly if you genuinely need it.

## References

- Bernstein, Hadzilacos, Goodman — *Concurrency Control and Recovery in Database Systems* (1987) — canonical treatment of 2PC and its blocking property
- Gray — *Notes on Database Operating Systems* (1978) — original formulation of 2PC
- Richardson — *Pattern: Saga*: https://microservices.io/patterns/data/saga.html
- Confluent — *Exactly-Once Semantics Are Possible: Here's How Apache Kafka Does It*: https://www.confluent.io/blog/exactly-once-semantics-are-possible-heres-how-apache-kafka-does-it/
- Kafka docs — *Transactions*: https://kafka.apache.org/documentation/#semantics

---

[← Previous: Saga Pattern](05-saga-pattern.md) | [Next: Consensus — Raft and Paxos →](07-consensus-raft-paxos.md) | [Back to index](README.md)
