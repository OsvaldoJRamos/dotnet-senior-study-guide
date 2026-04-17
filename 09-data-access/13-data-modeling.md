# Data Modeling

Data modeling is where most real performance problems are *created* — years before the slow query ticket is filed. This file covers the two families a senior .NET engineer crosses regularly: **relational normalization** and the main **NoSQL** modeling disciplines (document, key-value, wide-column).

## Relational: normalization

Normalization is a set of rules for eliminating redundancy in relational schemas. The early forms — 1NF → 3NF — come up in every interview.

| Form | Rule | What it eliminates |
|---|---|---|
| **1NF** | Atomic column values; no repeating groups | Multi-valued columns ("tags" as a comma-separated string) |
| **2NF** | 1NF + every non-key column depends on the **whole** primary key | Partial dependencies on composite keys |
| **3NF** | 2NF + non-key columns depend **only on the key** (no transitive dependencies) | Redundancy where a non-key column determines another non-key column |
| **BCNF** | Every non-trivial functional dependency has a superkey on the left | Edge cases 3NF misses (rare in practice) |

### Example walk-through

A denormalized `Orders` table:

| OrderId | CustomerId | CustomerName | CustomerCity | ProductId | ProductName | Qty |
|---|---|---|---|---|---|---|

**1NF** — already atomic, OK.

**2NF** — if `(OrderId, ProductId)` is the primary key, `CustomerName` depends only on `OrderId` (not on `ProductId`). Split:

```
Orders(OrderId PK, CustomerId, CustomerName, CustomerCity)
OrderLines(OrderId, ProductId, ProductName, Qty)
```

**3NF** — `CustomerName` and `CustomerCity` depend on `CustomerId`, not on `OrderId`. Split again:

```sql
CREATE TABLE Customers (
    CustomerId int PRIMARY KEY,
    CustomerName nvarchar(200) NOT NULL,
    CustomerCity nvarchar(100) NOT NULL
);

CREATE TABLE Orders (
    OrderId int PRIMARY KEY,
    CustomerId int NOT NULL REFERENCES Customers(CustomerId),
    OrderDate datetime2 NOT NULL
);

CREATE TABLE OrderLines (
    OrderId int NOT NULL REFERENCES Orders(OrderId),
    ProductId int NOT NULL REFERENCES Products(ProductId),
    Qty int NOT NULL,
    PRIMARY KEY (OrderId, ProductId)
);
```

### Denormalization trade-offs

Normalization removes redundancy — at the cost of more joins at query time. Denormalizing is a deliberate reversal when read performance matters more than write simplicity.

| Pattern | When it's right |
|---|---|
| **Duplicate a column** (e.g., copy `CustomerName` into `Orders`) | The source value rarely changes *and* the join is on a hot path |
| **Summary columns** (`Orders.TotalAmount`) | Recomputing from lines is expensive and the value is read often |
| **Materialized views / indexed views** | Aggregations that would otherwise scan millions of rows |
| **Read-side projections** (CQRS) | Different shapes for read vs write, kept in sync via events or CDC |

> The textbook advice — *"normalize until it hurts, denormalize until it works"* — survives because it's right. Start in 3NF; denormalize specific columns when you have a measured performance reason and a plan to keep them consistent.

## NoSQL: modeling by access pattern

The core mental shift: in a relational database, you design **the data**, and queries follow. In NoSQL, you design **the queries**, and the data follows.

AWS puts it directly for DynamoDB: *"In RDBMS, you design for flexibility without worrying about implementation details or performance... In DynamoDB, you design your schema specifically to make the most common and important queries as fast and as inexpensive as possible. Your data structures are tailored to the specific requirements of your business use cases."* (`docs.aws.amazon.com`)

## Document modeling (MongoDB, Cosmos DB, Postgres JSONB)

Two questions decide every relationship: **embed** or **reference**.

### Embed

Put the related data inside the parent document.

```json
{
  "_id": "order-42",
  "customer": { "id": "c-7", "name": "Ada" },
  "lines": [
    { "productId": "p-1", "name": "Keyboard", "qty": 1, "price": 120 },
    { "productId": "p-2", "name": "Mouse",    "qty": 1, "price":  40 }
  ]
}
```

Choose embed when:

- Children are **read together with the parent** almost always.
- Children are **owned** by the parent (no independent lifecycle).
- Total document size stays well under the per-document limit (16 MB in MongoDB).
- Cardinality is bounded (an order has ~5 lines, not ~50,000).

### Reference

Keep related data in its own collection and store an ID.

```json
{ "_id": "order-42", "customerId": "c-7", "lineIds": ["l-1", "l-2"] }
```

Choose reference when:

- The child is **shared** (a product referenced by many orders — embedding would duplicate it).
- The child has its own **independent lifecycle** (updated separately, queried on its own).
- Cardinality is unbounded or very large.

### The N+1 in document stores

MongoDB and friends support `$lookup` for joins — but joins in a document store are slower and less optimized than in a relational engine. **If you find yourself writing $lookup on every read, your data is in the wrong shape.** Either embed or denormalize.

### Polymorphic / schema-flexible collections

Document stores let you mix shapes in one collection (`type = "user" | "admin" | "bot"`). Useful; also the fastest way to a garbage dataset. Enforce shape at the application layer (validation, JSON Schema in MongoDB, Pydantic / FluentValidation in code), not "the driver will handle it".

## Key-value modeling (Redis, DynamoDB partition key)

The whole model is **one value per key**. Access patterns define the keys.

Common patterns:

- `user:{id}` → user profile JSON.
- `session:{token}` → session payload, with TTL.
- `cart:{userId}` → JSON of the cart.
- `rate-limit:{userId}:{endpoint}:{minute}` → counter with TTL.

Composite keys are the primary modeling tool. Want "all carts updated today"? A pure KV store can't scan — add a secondary index structure (Redis sorted set) or move to a store that supports range queries.

### Redis data types as modeling tools

| Type | Use for |
|---|---|
| **String** | Cached JSON, counters, feature flags |
| **Hash** | User profile fields (partial update) |
| **List** | FIFO queue, recent activity feed |
| **Set** | Tags, unique visitors |
| **Sorted Set (ZSET)** | Leaderboards, time-ordered indexes |
| **Stream** | Durable append-only log (producer/consumer with consumer groups) |

Picking the right Redis type can replace a whole layer of application logic.

## Wide-column modeling (Cassandra, ScyllaDB)

Cassandra is the canonical wide-column store. The primary key is **composite**: `(partition key, clustering column 1, clustering column 2, ...)`.

The Cassandra project describes the primary key as *"a collection of columns identified by a unique primary key made up of the partition key and optionally additional clustering keys."* (`cassandra.apache.org`)

- **Partition key** — mandatory; decides which node holds the row.
- **Clustering columns** — optional; define the **order** of rows within a partition.

### Query-first design

You don't pick a partition key *after* listing your tables. You pick it **per query**:

1. Write the top N queries the application will run.
2. For each query, design a table whose `(partition_key, clustering_columns)` makes it an efficient single-partition read.
3. Duplicate data across tables as needed — **denormalization is the default**, not the exception.

### Example

Query: *"latest 20 messages in chat X, ordered by time descending"*.

```sql
CREATE TABLE messages_by_chat (
    chat_id uuid,
    sent_at timestamp,
    message_id uuid,
    sender uuid,
    body text,
    PRIMARY KEY (chat_id, sent_at, message_id)
) WITH CLUSTERING ORDER BY (sent_at DESC, message_id ASC);

SELECT * FROM messages_by_chat
WHERE chat_id = ? LIMIT 20;
```

Single-partition read, pre-sorted. Adding another query ("all messages sent by user Y") means a *different* table, often with the user as the partition key — not an `ALLOW FILTERING` hack.

### Partition size

Keep partitions bounded. A widely cited community guideline is to target well under ~100 MB or ~100,000 rows per partition (not a specific number from the Apache Cassandra docs, but repeated in DataStax guidance and operator experience). Unbounded partitions (one partition per chat forever) become hotspots and slow compactions. Include a time bucket in the partition key to cap growth:

```sql
PRIMARY KEY ((chat_id, year_month), sent_at, message_id)
```

## DynamoDB single-table design

DynamoDB pushes query-first modeling to its logical conclusion: **most applications should use one table, not one table per entity**.

AWS: *"You should maintain as few tables as possible in a DynamoDB application. Having fewer tables keeps things more scalable, requires less permissions management, and reduces overhead for your DynamoDB application."* (`docs.aws.amazon.com`)

### The pattern

One table, two generic keys (`PK`, `SK`), multiple entity types sharing them with structured values:

```
PK                  SK                      Attributes
-----------------   ---------------------   ------------------------------
USER#42             PROFILE                 name=Ada, email=...
USER#42             ORDER#2024-04-16#999    total=120, status=Paid
USER#42             ORDER#2024-04-17#1001   total=40,  status=Pending
ORDER#999           ITEM#p-1                qty=1, price=120
ORDER#999           ITEM#p-2                qty=1, price=40
```

Queries:

- `PK = USER#42 AND begins_with(SK, "ORDER#")` → all orders for a user, sorted by date.
- `PK = USER#42 AND SK = PROFILE` → user profile.
- `PK = ORDER#999 AND begins_with(SK, "ITEM#")` → all items in an order.

Add **Global Secondary Indexes (GSIs)** with inverted or alternative keys for additional access patterns.

AWS: *"You shouldn't start designing your schema for DynamoDB until you know the questions it will need to answer. Understanding the business problems and the application use cases up front is essential."* Single-table design is hostile to "figure it out later" — which is exactly why it's fast.

### When to keep multiple tables

- Very different access-control boundaries (compliance).
- Wildly different retention/TTL needs.
- Different sharing characteristics (one hot entity shouldn't noisily-neighbor a cold one).

Single-table is the default; multiple tables are a conscious exception.

## Relational vs NoSQL modeling at a glance

| Axis | Relational | NoSQL (document / KV / wide-column) |
|---|---|---|
| **Design order** | Data first, queries follow | Queries first, data follows |
| **Redundancy** | Minimized (3NF) | Expected (denormalize for read) |
| **Joins** | First-class, cheap on indexed columns | Absent or expensive; model to avoid them |
| **Schema changes** | `ALTER TABLE`, migrations | Application-level versioning; data with mixed shapes |
| **Constraints** | FKs, checks, uniqueness enforced by engine | Mostly application-level |
| **Scaling** | Vertical first; sharding is hard | Horizontal by design |

## Pitfalls

- **Over-normalizing a read-heavy OLTP table.** Every query joins seven tables. Consider a denormalized read model — either a column on the main table or a CQRS read side.
- **Under-normalizing and calling it "NoSQL thinking" in Postgres.** JSONB has its place (flexible payloads), not "dump the whole aggregate as JSON because it's easier". You lose query power and indexing clarity.
- **Embedding unbounded children.** A document's array that grows forever eventually hits the document size limit and corrupts assumptions.
- **Unbounded partition keys in Cassandra/DynamoDB.** Hot partition, throttling, compaction pain. Time-bucket it.
- **Tables-per-entity in DynamoDB.** Joins across tables require multiple round-trips and lose the benefit of the model. Go single-table unless you have a real reason not to.
- **Modeling NoSQL as if it were SQL.** References everywhere, `$lookup` in every read, application-level joins. If the access pattern needs joins, use a relational database.

## Rule of thumb

> Normalize first in relational systems; embed/denormalize first in document and wide-column systems. Always design for the top 5 access patterns explicitly — everything else is secondary.

---

[← Previous: Change Data Capture](12-change-data-capture.md) | [Back to index](README.md)
