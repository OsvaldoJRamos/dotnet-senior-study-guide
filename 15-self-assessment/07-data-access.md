# Data Access

> Read the questions, think about your answer, then click to reveal.

---

### 1. What is the difference between an ORM, a Micro ORM, and raw ADO.NET? When would you pick each?

<details>
<summary>Reveal answer</summary>

| Approach | Example | Strengths | Best for |
|----------|---------|-----------|----------|
| **ORM** | EF Core | Change tracking, migrations, LINQ, relationships | Complex domain models, rapid development |
| **Micro ORM** | Dapper | Fast, maps SQL to objects, minimal overhead | Read-heavy, performance-critical queries |
| **ADO.NET** | SqlConnection | Full control, no abstraction | Bulk operations, stored procs, extreme perf needs |

Choose **EF Core** when productivity matters and the domain is complex. Choose **Dapper** for high-throughput reads or when you want full SQL control. Choose **ADO.NET** for edge cases like bulk inserts or very low-level data access.

Deep dive: [ORM vs Micro ORM vs ADO.NET](../09-data-access/01-orm-vs-microorm-vs-adonet.md)

</details>

---

### 2. What is the N+1 problem in Entity Framework, and how do you fix it?

<details>
<summary>Reveal answer</summary>

The **N+1 problem** occurs when EF loads a parent entity (1 query) and then lazily loads each child entity individually (N queries). For 100 orders with their items, that's 101 queries instead of 1-2.

Fixes:
- **Eager loading**: `.Include(o => o.Items)` -- generates a JOIN in a single query.
- **Explicit loading**: `context.Entry(order).Collection(o => o.Items).LoadAsync()` -- controlled, still extra queries.
- **Projection**: `Select` only the fields you need into a DTO -- avoids loading full entities.
- **Split queries**: `.AsSplitQuery()` -- separate queries per collection, avoids cartesian explosion.

Deep dive: [Entity Framework](../09-data-access/02-entity-framework.md)

</details>

---

### 3. What is the difference between Lazy Loading and Eager Loading in EF Core?

<details>
<summary>Reveal answer</summary>

- **Lazy Loading** -- related entities are loaded **on first access** via virtual navigation properties and a proxy. Convenient but dangerous: can cause N+1 silently and fails outside the DbContext scope.
- **Eager Loading** -- related entities are loaded **upfront** with `.Include()`. You get all data in one (or a few) queries, making behavior predictable.

**Best practice**: prefer eager loading or projections. If lazy loading is used, be very aware of where data access happens to avoid performance surprises.

Deep dive: [Entity Framework](../09-data-access/02-entity-framework.md)

</details>

---

### 4. How does the EF Core Change Tracker work, and when should you use `AsNoTracking()`?

<details>
<summary>Reveal answer</summary>

The **Change Tracker** keeps a snapshot of every entity queried from the database. On `SaveChangesAsync()`, it compares current values to the snapshot and generates `INSERT`/`UPDATE`/`DELETE` SQL.

Use **`AsNoTracking()`** for **read-only queries** -- it skips snapshot creation, reducing memory and improving performance. Since there's no tracking, you can't call `SaveChanges()` to persist modifications on those entities.

```csharp
var products = await context.Products
    .AsNoTracking()
    .Where(p => p.IsActive)
    .ToListAsync();
```

Deep dive: [Entity Framework](../09-data-access/02-entity-framework.md)

</details>

---

### 5. How do EF Core migrations work? What's the difference between `Add-Migration` and `Update-Database`?

<details>
<summary>Reveal answer</summary>

Migrations let you **version-control your database schema** alongside your code.

- **`Add-Migration <Name>`** -- compares the current model to the last snapshot and generates a C# migration file with `Up()` and `Down()` methods.
- **`Update-Database`** -- applies pending migrations to the database by executing the `Up()` methods in order.

In production, you typically generate a **SQL script** (`Script-Migration`) or use a migration bundle instead of running `Update-Database` directly for safer, reviewable deployments.

Deep dive: [Entity Framework](../09-data-access/02-entity-framework.md)

</details>

---

### 6. When would you choose Dapper over EF Core?

<details>
<summary>Reveal answer</summary>

Choose **Dapper** when:
- You need **raw performance** on read-heavy paths (Dapper maps results ~2-3x faster than EF Core with tracking).
- You already have complex, hand-tuned **SQL or stored procedures**.
- You want a **thin layer** without change tracking, migrations, or LINQ translation.

Choose **EF Core** when you benefit from change tracking, migrations, LINQ, and relationships. Many teams use **both together** -- EF Core for writes and complex domain logic, Dapper for high-performance reads.

Deep dive: [ORM vs Micro ORM vs ADO.NET](../09-data-access/01-orm-vs-microorm-vs-adonet.md)

</details>

---

### 7. What are database indexes and how do they improve query performance? What's the trade-off?

<details>
<summary>Reveal answer</summary>

An **index** is a sorted data structure (typically B-tree) that lets the database find rows without scanning the entire table.

**Benefits**: dramatically faster `WHERE`, `JOIN`, and `ORDER BY` on indexed columns.

**Trade-offs**:
- **Slower writes** -- every `INSERT`/`UPDATE`/`DELETE` must also update the index.
- **Storage overhead** -- indexes consume disk space.
- **Maintenance** -- fragmented indexes need periodic rebuilding.

**Rule of thumb**: index columns you frequently filter or join on; avoid indexing columns with low selectivity (e.g., boolean flags).

Deep dive: [Query Optimization](../09-data-access/04-query-optimization.md)

</details>

---

### 8. What is "parameter sniffing" and when does it become a problem?

<details>
<summary>Reveal answer</summary>

**Parameter sniffing** is when SQL Server creates an execution plan based on the **first parameter values** it sees and reuses that plan for all subsequent calls. This is usually good -- plan reuse is efficient.

It becomes a problem when the first values are **atypical** (e.g., a value that matches 1 row vs. another that matches 1 million rows). The cached plan is optimal for the first case but terrible for the second.

Mitigations: `OPTION (RECOMPILE)`, `OPTIMIZE FOR UNKNOWN`, query hints, or plan guides.

Deep dive: [Query Optimization](../09-data-access/04-query-optimization.md)

</details>

---

### 9. What does "sargable" mean, and why should you avoid wrapping columns in functions in a WHERE clause?

<details>
<summary>Reveal answer</summary>

**SARGable** = "Search ARGument able." A predicate is sargable when the database can use an **index seek** to evaluate it.

Wrapping a column in a function (e.g., `WHERE YEAR(CreatedDate) = 2025`) prevents the optimizer from using the index on `CreatedDate` because it must evaluate the function for every row (index scan).

**Fix**: rewrite to a range predicate:

```sql
-- Non-sargable
WHERE YEAR(CreatedDate) = 2025

-- Sargable
WHERE CreatedDate >= '2025-01-01' AND CreatedDate < '2026-01-01'
```

Deep dive: [Query Optimization](../09-data-access/04-query-optimization.md)

</details>

---

### 10. When would you choose a NoSQL database over a relational (SQL) database?

<details>
<summary>Reveal answer</summary>

| Choose SQL when | Choose NoSQL when |
|-----------------|-------------------|
| Data has clear relationships and needs referential integrity | Data is denormalized or document-shaped |
| Transactions with ACID guarantees are required | Horizontal scalability is a priority |
| Schema is well-known and stable | Schema evolves rapidly |
| Complex joins and aggregations | High write throughput, eventual consistency is OK |

Examples: **MongoDB** (documents), **Redis** (key-value/cache), **Cassandra** (wide-column, massive write throughput). Many systems use **both** -- SQL for transactional data, NoSQL for caching, logs, or search.

Deep dive: [Databases](../09-data-access/03-databases.md)

</details>

---

### 11. What is connection pooling and why is it important?

<details>
<summary>Reveal answer</summary>

**Connection pooling** reuses database connections instead of creating a new one for every request. Opening a connection is expensive (TCP handshake, authentication, TLS). The pool maintains a set of open connections that are checked out and returned.

In .NET, `SqlConnection` pooling is **on by default** (controlled by the connection string). Key settings: `Max Pool Size`, `Min Pool Size`, `Connection Lifetime`.

**Pitfall**: if you don't dispose connections (`using` / `await using`), they aren't returned to the pool, eventually exhausting it and causing timeouts.

Deep dive: [ORM vs Micro ORM vs ADO.NET](../09-data-access/01-orm-vs-microorm-vs-adonet.md)

</details>

---

### 12. What are the standard SQL transaction isolation levels, and what problems does each prevent?

<details>
<summary>Reveal answer</summary>

| Isolation Level | Dirty Read | Non-repeatable Read | Phantom Read |
|----------------|-----------|-------------------|-------------|
| Read Uncommitted | Possible | Possible | Possible |
| **Read Committed** (default) | Prevented | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible |
| Serializable | Prevented | Prevented | Prevented |

- **Dirty read** -- reading uncommitted changes from another transaction.
- **Non-repeatable read** -- re-reading a row yields different values because another transaction modified it.
- **Phantom read** -- re-running a query returns new rows inserted by another transaction.

Higher isolation = more correctness but more locking and lower throughput.

Deep dive: [Databases](../09-data-access/03-databases.md)

</details>

---
