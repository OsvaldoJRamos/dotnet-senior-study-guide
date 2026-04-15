# Messaging (Message Brokers)

## What it is

Messaging is the **asynchronous** communication pattern between services using an intermediary (broker). The producer sends the message and **does not wait** for a response.

```
[Producer] ──publishes──→ [Broker] ──delivers──→ [Consumer]
```

## Why use it

- **Decoupling**: the producer does not need to know who consumes
- **Resilience**: if the consumer is down, the message stays in the queue
- **Scale**: multiple consumers process in parallel
- **Load spikes**: the queue absorbs the spike, consumers process at their own pace

## Messaging patterns

### 1. Queue (point-to-point)

A message is processed by **a single consumer**:

```
[Producer] → [Queue] → [Consumer 1]
                      → [Consumer 2]  (only one receives each message)
```

Use cases: order processing, email sending, asynchronous jobs.

### 2. Pub/Sub (publish/subscribe)

A message is delivered to **all subscribers**:

```
[Publisher] → [Topic] → [Subscriber 1] (receives)
                      → [Subscriber 2] (receives)
                      → [Subscriber 3] (receives)
```

Use cases: notifications, synchronization between services, event-driven architecture.

## RabbitMQ

The most popular message broker for .NET applications. Based on the **AMQP** protocol.

### Concepts

- **Exchange**: receives messages from the producer and routes them to queues
- **Queue**: stores messages until they are consumed
- **Binding**: rule that connects an exchange to a queue
- **Routing Key**: key used for routing

```
[Producer] → [Exchange] ──routing key──→ [Queue] → [Consumer]
```

### Exchange Types

| Type | Routing |
|------|------------|
| **Direct** | Exact routing key |
| **Fanout** | All queues (broadcast) |
| **Topic** | Routing key with wildcards (`order.*`, `#.error`) |
| **Headers** | Message headers |

### Example with MassTransit (.NET)

```csharp
// Configuration
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<OrderCreatedConsumer>();

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("rabbitmq://localhost");
        cfg.ConfigureEndpoints(context);
    });
});

// Publish event
public class OrderService
{
    private readonly IPublishEndpoint _publish;

    public async Task CreateOrder(Order order)
    {
        // ... save order ...
        await _publish.Publish(new OrderCreatedEvent(order.Id, order.Total));
    }
}

// Consume event
public class OrderCreatedConsumer : IConsumer<OrderCreatedEvent>
{
    public async Task Consume(ConsumeContext<OrderCreatedEvent> context)
    {
        var message = context.Message;
        // ... send email, update inventory, etc.
    }
}
```

## Apache Kafka

An **event streaming** platform. Different from traditional queues:

- Messages are **persisted** (distributed log)
- Consumers control their **offset** (can re-read messages)
- Very high throughput (millions of messages per second)

### Concepts

- **Topic**: message category
- **Partition**: subdivision of a topic (parallelism)
- **Consumer Group**: group of consumers that share partitions
- **Offset**: consumer's position in the log

### Kafka vs RabbitMQ

| Aspect | RabbitMQ | Kafka |
|---------|----------|-------|
| Model | Message broker (queues) | Event streaming (log) |
| Consumed message | Removed from queue (classic); retained in log (Streams) | Stays in the log |
| Replay full history | No for classic queues; Yes with RabbitMQ Streams (since 3.9, July 2021) | Yes (by offset) |
| Throughput | Medium-high | Very high |
| Latency | Lower | Higher |
| Ideal use | Async tasks, RPC | Event sourcing, analytics, high scale |
| Complexity | Lower | Higher |

> **On "re-reading" in RabbitMQ:** classic queues do support **requeue via `nack`**, **dead-letter exchanges**, and consumer retries — what they don't support is replaying the full history after a message has been acknowledged. For that use case, **RabbitMQ Streams** (since 3.9, July 2021) provide a Kafka-like append-only log with persistent, offset-based replay.

## Azure Service Bus

Microsoft's managed service. Supports **queues** and **topics/subscriptions**:

```csharp
// Send message
var client = new ServiceBusClient(connectionString);
var sender = client.CreateSender("orders-queue");

await sender.SendMessageAsync(new ServiceBusMessage(
    JsonSerializer.Serialize(order)));

// Receive message
var processor = client.CreateProcessor("orders-queue");
processor.ProcessMessageAsync += async args =>
{
    var order = JsonSerializer.Deserialize<Order>(args.Message.Body.ToString());
    await args.CompleteMessageAsync(args.Message);
};
await processor.StartProcessingAsync();
```

## Delivery guarantees

| Guarantee | Description | When to use |
|----------|-----------|-------------|
| **At-most-once** | May lose, never duplicates | Logs, metrics |
| **At-least-once** | Never loses, may duplicate | Most cases (with idempotency) |
| **Exactly-once** | Never loses, never duplicates | Financial transactions (expensive/complex) |

> In practice, use **at-least-once** with **idempotent consumers**.

> **On "exactly-once":** true end-to-end exactly-once delivery is **impossible** in a distributed system (the Two Generals Problem). What is achievable is **effectively-once** processing: at-least-once delivery combined with idempotent consumers (idempotency keys on commands, dedup tables on message IDs, or upserts on natural keys). Kafka's "exactly-once semantics" (EOS) is **within-cluster transactional processing** — it covers `consume → process → produce` inside the same Kafka cluster, not end-to-end delivery to external systems.

## Poison messages and dead-letter queues

A **poison message** is one that keeps failing — a parse error, a schema mismatch, a violated invariant — no matter how many times you retry. Without protection it blocks the queue and burns CPU on endless retries.

The standard mitigation:

1. **Cap retries** per message (e.g., 5 attempts, with exponential backoff).
2. After the cap, move the message to a **Dead-Letter Queue (DLQ)** for manual inspection instead of discarding it.
3. Alert on DLQ depth; triage, fix the root cause, then **replay** from the DLQ once the bug is fixed.

RabbitMQ supports this via **dead-letter exchanges** (`x-dead-letter-exchange` on the queue). Azure Service Bus has built-in DLQs per queue/subscription. Kafka does not have DLQs natively — teams usually implement them as a separate `*.dlq` topic managed by the consumer.

---

[← Previous: Microservices](09-microservices.md) | [Next: Idempotency and Race Conditions →](11-idempotency-and-race-conditions.md) | [Back to index](README.md)
