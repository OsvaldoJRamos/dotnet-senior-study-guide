# 17 - Distributed Systems

The foundational theory behind microservices, event-driven architectures, and cloud-native systems. This section covers the hard limits distributed systems operate under (CAP, consensus, ordering) and the patterns seniors use to ship correct systems despite those limits (outbox, saga, idempotency).

## Contents

1. [CAP Theorem](01-cap-theorem.md) - Gilbert/Lynch 2002 formal statement, PACELC extension, what "pick two" actually means
2. [Consistency Models](02-consistency-models.md) - Strong, sequential, causal, eventual, and session guarantees — with .NET examples
3. [Idempotency](03-idempotency.md) - Why retries make idempotency mandatory, keys, at-least-once vs exactly-once, HTTP semantics
4. [Outbox Pattern](04-outbox-pattern.md) - The dual-write problem and how Wolverine, MassTransit, and CDC-based relays solve it
5. [Saga Pattern](05-saga-pattern.md) - Choreography vs orchestration, compensating transactions, MassTransit state machines
6. [Distributed Transactions](06-distributed-transactions.md) - Why 2PC is avoided, sagas vs outbox, Kafka exactly-once semantics
7. [Consensus — Raft and Paxos](07-consensus-raft-paxos.md) - Why consensus is needed, Raft leader election and log replication, etcd and Consul usage
8. [Clocks and Ordering](08-clocks-and-ordering.md) - Physical vs logical clocks, Lamport timestamps, vector clocks, hybrid logical clocks

## Useful Links

- [Async Processing and Message Queues (Hookdeck)](https://medium.com/hookdeck/an-introduction-to-asynchronous-processing-and-message-queues-218af596bf1b) — primer on async processing, queue semantics, and decoupling.
- [Message Queues in System Design (GeeksforGeeks)](https://www.geeksforgeeks.org/system-design/message-queues-system-design/) — queue fundamentals, producer/consumer, ordering, delivery guarantees.
- [Event-Driven Architecture (GeeksforGeeks)](https://www.geeksforgeeks.org/system-design/event-driven-architecture-system-design/) — EDA building blocks, event broker vs mediator, choreography vs orchestration.
- [ACID vs BASE (AWS)](https://aws.amazon.com/compare/the-difference-between-acid-and-base-database/) — side-by-side comparison and trade-offs.
- [CAP Theorem (GeeksforGeeks)](https://www.geeksforgeeks.org/dbms/the-cap-theorem-in-dbms/) — additional perspective on CAP, complementary to Gilbert/Lynch's paper.
- [Consistency in System Design (GeeksforGeeks)](https://www.geeksforgeeks.org/system-design/consistency-in-system-design/) — strong/eventual/causal consistency walkthrough.
- [Data Replication: A Key Component (ByteByteGo)](https://blog.bytebytego.com/p/data-replication-a-key-component) — sync vs async replication, leader-follower, multi-leader, leaderless.
- [Fallacies of Distributed Computing (Wikipedia)](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing) — Deutsch's eight assumptions that every distributed-systems failure traces back to.

---

[Back to index](../README.md)
