# Saga Pattern

## The problem

A business operation spans **multiple services**, each with its own database. "Place order" requires:

1. Reserve payment (Payments service).
2. Reserve stock (Inventory service).
3. Create shipping label (Shipping service).
4. Send confirmation email (Notifications service).

A single ACID transaction across four services is [not viable](06-distributed-transactions.md) in modern microservices. You need a way to keep the overall business invariant ("an order is committed only if all its steps committed") without a distributed transaction.

## The pattern

Hector Garcia-Molina and Kenneth Salem introduced **sagas** in 1987 ("Sagas", ACM SIGMOD) as long-running business transactions composed of a sequence of **local transactions**:

- Each step is an ACID transaction in one service.
- After a step succeeds, a message triggers the next step.
- If a step fails, previously completed steps are undone by running **compensating transactions** — business-level reversals ("refund the payment", "release reserved stock"), not database rollbacks.

Chris Richardson's concise definition: *"a sequence of local transactions. Each local transaction updates the database and publishes a message or event to trigger the next local transaction in the saga."*

## Choreography vs orchestration

Two coordination styles. Both are valid; pick based on complexity.

### Choreography

No central coordinator. Each service **publishes events** and **subscribes** to events from other services. The business flow emerges from the reactions.

```text
OrderService ──OrderCreated──▶  PaymentService
                                    │
                                    └─PaymentReserved──▶ InventoryService
                                                              │
                                                              └─StockReserved──▶ ShippingService

On failure (e.g., StockReservationFailed):
                            ◀──StockReservationFailed── InventoryService
PaymentService ◀── compensates (refund)
OrderService   ◀── compensates (cancel order)
```

**Pros**

- No single point of failure; services are autonomous.
- Easy to add a new participant — just subscribe to the right events.
- Natural fit with a real event bus.

**Cons**

- Flow is implicit — you can't look at one class and know the whole saga. "Where does `StockReserved` go?" is answered only by grep.
- Cycles and race conditions are easy to introduce.
- Debugging requires distributed tracing.

> Choreography works best when the flow is short (2–3 steps) and each participant's role is obvious.

### Orchestration

A central **orchestrator** (the saga itself) sends explicit commands to participants and reacts to their replies.

```text
                ┌──────────────┐
                │ Orchestrator │
                └──────┬───────┘
                       │
       ┌───────────────┼────────────────┐
       ▼               ▼                ▼
  ReservePayment   ReserveStock   CreateShipment
  (Payments)       (Inventory)    (Shipping)

On failure: the orchestrator issues compensations to previously completed steps.
```

**Pros**

- Flow is explicit and readable in one place (usually a state machine).
- Easier to reason about, test, and monitor.
- Timeouts and retries are centralized.

**Cons**

- Orchestrator is a kind of coupling — adding a new step means changing it.
- Orchestrator must be highly available; if it dies mid-saga, progress stalls.

> Orchestration wins as saga length grows (4+ steps) or when the flow has branches and error paths.

## Compensating transactions — the hard part

A compensation is **not** a database rollback. It's a business-semantic reversal of a completed transaction.

| Forward step | Compensation |
|---|---|
| `ReservePayment($100)` | `RefundPayment($100)` |
| `ReserveStock(sku, qty=3)` | `ReleaseStock(sku, qty=3)` |
| `CreateShipment(orderId)` | `CancelShipment(orderId)` |
| `SendWelcomeEmail(user)` | *Cannot un-send* — design the step to be cancelable or idempotent-resendable |

Rules:

1. **Compensations must be idempotent.** Messages get redelivered; the compensation will run more than once.
2. **Compensations must always succeed eventually.** If they can't, the system is stuck in an inconsistent state. Retry with backoff; escalate to humans after a threshold.
3. **Some actions are uncompensable** (send email, ship physical goods). Either design the step to be reversible (soft-send with a cancel window), or put it *last* in the saga so nothing downstream can fail.

## Isolation is weaker than ACID

Chris Richardson is blunt: sagas **lack isolation**. Concurrent sagas can see each other's partial state. Example:

1. Saga A reserves stock for SKU-42.
2. Saga B queries "is SKU-42 in stock?" — sees the reserved state as unavailable.
3. Saga A's next step fails, compensation releases the stock.
4. Saga B has already rejected the user.

Countermeasures (Richardson's terminology):

- **Semantic lock** — mark the record as "pending" so other sagas know not to act on it.
- **Commutative updates** — design operations to be order-independent (e.g., debit/credit instead of set-balance).
- **Pessimistic view** — structure the flow so the user only sees confirmed data.
- **Reread value** — before acting on cached state, reread from the source of truth.
- **Version file** — record every state change so the final consistent state can be reconstructed.
- **By value** — use business rules to decide isolation strategy based on the data's criticality.

## MassTransit — State Machine Sagas (Automatonymous)

MassTransit recommends **saga state machines** over consumer sagas. A state machine defines states, events, and transitions in one place.

```csharp
public class OrderSaga : SagaStateMachineInstance
{
    public Guid CorrelationId { get; set; }
    public string CurrentState { get; set; } = null!;
    public decimal Amount { get; set; }
}

public class OrderStateMachine : MassTransitStateMachine<OrderSaga>
{
    public State AwaitingPayment { get; private set; } = null!;
    public State AwaitingStock { get; private set; } = null!;
    public State Shipped { get; private set; } = null!;

    public Event<OrderCreated> OrderCreated { get; private set; } = null!;
    public Event<PaymentReserved> PaymentReserved { get; private set; } = null!;
    public Event<StockReservationFailed> StockReservationFailed { get; private set; } = null!;

    public OrderStateMachine()
    {
        InstanceState(x => x.CurrentState);

        Event(() => OrderCreated, x => x.CorrelateById(m => m.Message.OrderId));

        Initially(
            When(OrderCreated)
                .Then(ctx => ctx.Saga.Amount = ctx.Message.Amount)
                .Publish(ctx => new ReservePayment(ctx.Saga.CorrelationId, ctx.Saga.Amount))
                .TransitionTo(AwaitingPayment));

        During(AwaitingPayment,
            When(PaymentReserved)
                .Publish(ctx => new ReserveStock(ctx.Saga.CorrelationId))
                .TransitionTo(AwaitingStock));

        During(AwaitingStock,
            When(StockReservationFailed)
                .Publish(ctx => new RefundPayment(ctx.Saga.CorrelationId, ctx.Saga.Amount))  // compensation
                .Finalize());

        SetCompletedWhenFinalized();
    }
}
```

Storage options per the MassTransit docs include Entity Framework, Dapper, NHibernate, MongoDB, Marten, Azure Cosmos DB, Redis, DynamoDB, and Azure Table Storage. Correlation uses a `CorrelationId` (GUID) — all events for the same saga share it.

**Concurrency**: MassTransit recommends **optimistic concurrency** for nearly all scenarios — if two messages hit the same saga instance simultaneously, one wins and the other retries via a concurrency-exception retry policy. Pessimistic options (serializable transactions / row locks, where the store supports them) are available for pathological cases. Running an in-memory outbox inside the saga buffers outbound messages until the saga state commits; without it, a crash between "send command" and "save state" can re-send.

## NServiceBus

NServiceBus has built-in sagas. The data model is similar — a saga class with state, correlated via configured properties, persisted through SQL, NHibernate, or other supported stores. Verify against the current NServiceBus docs when implementing, as API conventions evolve per major version.

## Failure scenarios to think through

| Scenario | Handling |
|---|---|
| Step 3 fails after steps 1 and 2 committed | Orchestrator emits compensations for steps 2 and 1. |
| Compensation itself fails | Retry with backoff. If terminal, park on a dead-letter and alert. |
| Orchestrator crashes mid-saga | State is persisted; on restart, the saga resumes from the last saved state. |
| Message arrives twice | Consumer is idempotent; the saga's state machine only accepts the message once per state. |
| Message arrives out of order | State machine rejects events not valid in the current state (e.g., `PaymentReserved` while `Finalized`). |
| Forward step times out | Treat timeout as failure, run compensations for completed steps. |

## When NOT to use sagas

- Operation fits in a single service / single database. Use an ACID transaction.
- All steps are read-only or naturally idempotent retries. A retry loop is simpler than a saga.
- The business can tolerate "fire and forget" — sagas are for operations with business invariants that span services.

## References

- Garcia-Molina, Salem — *Sagas* (ACM SIGMOD, 1987)
- Richardson — *Pattern: Saga*: https://microservices.io/patterns/data/saga.html
- MassTransit — *Sagas*: https://masstransit.io/documentation/patterns/saga
- MassTransit — *Saga State Machines*

---

[← Previous: Outbox Pattern](04-outbox-pattern.md) | [Next: Distributed Transactions →](06-distributed-transactions.md) | [Back to index](README.md)
