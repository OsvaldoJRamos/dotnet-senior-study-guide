# 16 - Distributed Systems

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

---

[Back to index](../README.md)
