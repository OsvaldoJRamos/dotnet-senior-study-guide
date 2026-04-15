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
| **Read Committed** | Prevented | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible |
| Serializable | Prevented | Prevented | Prevented |

- **Dirty read** -- reading uncommitted changes from another transaction.
- **Non-repeatable read** -- re-reading a row yields different values because another transaction modified it.
- **Phantom read** -- re-running a query returns new rows inserted by another transaction.

**Defaults vary by engine**: Read Committed is the default in SQL Server and PostgreSQL. **MySQL InnoDB defaults to Repeatable Read** (with gap locks that also prevent phantoms). Oracle defaults to Read Committed and has no Read Uncommitted level.

Higher isolation = more correctness but more locking and lower throughput.

Deep dive: [Transaction Isolation Levels](../09-data-access/06-transaction-isolation-levels.md)

</details>

---

### 13. What is the difference between a clustered and a non-clustered index?

<details>
<summary>Reveal answer</summary>

- **Clustered index** — defines the **physical order** of rows. The leaf pages of the index *are* the table rows. A table can have **only one**. The primary key becomes the clustered index by default.
- **Non-clustered index** — a separate structure whose leaf pages store the indexed columns and a **pointer to the row** (the clustered key, or the RID if the table is a heap). A table can have **many**.

When a non-clustered index doesn't contain all columns the query needs, a **key lookup** occurs — extra I/O to fetch missing columns from the clustered index. A **covering index** (using `INCLUDE`) eliminates this by storing the extra columns at the leaf level.

Deep dive: [SQL Indexes](../09-data-access/05-sql-indexes.md)

</details>

---

### 14. What's the difference between an Index Seek, an Index Scan, and a Table Scan?

<details>
<summary>Reveal answer</summary>

| Operator | Cost | What happens |
|----------|------|--------------|
| **Index Seek** | Low | B-tree navigation jumps straight to matching rows |
| **Index Scan** | Medium | Reads every leaf page of the index (predicate isn't usable or too many rows estimated) |
| **Table Scan** / **Clustered Index Scan** | High | Reads every row of the table |

Common causes of an unexpected scan despite an index existing:
- **Non-sargable predicate**: `WHERE YEAR(CreatedAt) = 2026` — function on the column disables the seek.
- **Implicit type conversion**: `varchar` column compared against `nvarchar` parameter.
- **Leading column not in the WHERE** for a composite index.
- **Stale statistics** causing bad cardinality estimates.
- `SELECT *` forcing many key lookups, making a scan look cheaper to the optimizer.

Deep dive: [SQL Indexes](../09-data-access/05-sql-indexes.md)

</details>

---

### 15. What is a covering index and when is it worth creating one?

<details>
<summary>Reveal answer</summary>

A **covering index** contains every column the query reads — either as index keys or via `INCLUDE`. The query is served entirely from the index, with no key lookup into the clustered index.

```sql
CREATE NONCLUSTERED INDEX IX_Orders_Customer_Covering
ON Orders(CustomerId)
INCLUDE (OrderDate, Total);

-- No key lookup needed
SELECT OrderDate, Total FROM Orders WHERE CustomerId = 42;
```

Create one when:
- A hot query runs frequently and shows a **key lookup** as a significant cost in the execution plan.
- The extra included columns aren't huge (covering indexes bloat the index size and slow writes).

Don't add `INCLUDE` blindly — monitor `sys.dm_db_index_usage_stats` and remove unused covering indexes.

Deep dive: [SQL Indexes](../09-data-access/05-sql-indexes.md)

</details>

---

### 16. What is Snapshot isolation and how is it different from Serializable?

<details>
<summary>Reveal answer</summary>

**Snapshot** is a **row-versioning** (MVCC) model, not a locking one. Each transaction sees a consistent picture of the database as of the moment it started. Writers don't block readers and vice versa; writer-vs-writer conflicts surface as **update conflict** errors (3960 in SQL Server).

**Serializable** uses **range locks** to make transactions behave as if they ran one at a time — preventing all anomalies (dirty, non-repeatable, phantoms) but increasing blocking and deadlock risk.

Use snapshot when readers are being blocked by writers and you can tolerate retries on update conflicts. Use serializable when you need strict "nothing new can appear" semantics in short, focused transactions.

Deep dive: [Transaction Isolation Levels](../09-data-access/06-transaction-isolation-levels.md)

</details>

---

### 17. What are the different lock granularities and modes in SQL Server?

<details>
<summary>Reveal answer</summary>

**Granularity** (smaller = more concurrency, more overhead):

`RID` (row in heap) → `KEY` (row in B-tree) → `PAGE` (8 KB) → `EXTENT` (8 pages) → `HoBT` → `TABLE` → `DATABASE`. Plus `XACT` for optimized locking (2022+).

**Modes** and what they mean:

| Mode | Purpose |
|---|---|
| **S** (Shared) | Reads |
| **U** (Update) | "I'll probably update" — blocks other U/X; prevents conversion deadlocks |
| **X** (Exclusive) | Writes; incompatible with everything |
| **IS / IX / SIX** | Intent locks at higher levels — signal lower-level S/X |
| **Sch-S / Sch-M** | Schema stability / modification (DDL) |
| **Range*** | Key-range locks for `SERIALIZABLE` to prevent phantoms |

**Key facts for an interview:**
- X is compatible with nothing.
- U+U is **not** compatible (that's the point — it prevents lost updates).
- Every query — even `NOLOCK` — takes `Sch-S`, which is why DDL can block everything.

Deep dive: [Database Locks](../09-data-access/07-database-locks.md)

</details>

---

### 18. What is lock escalation and how do you handle it?

<details>
<summary>Reveal answer</summary>

SQL Server collapses many fine-grained locks into a single higher-level lock when a statement acquires roughly **5,000 locks on one table/partition**, or when lock memory exceeds internal thresholds (~24% of DB engine memory). It's re-checked every ~1,250 additional locks.

**Why it exists:** tracking millions of row locks is expensive.

**Why it hurts:** a big `UPDATE` that should affect 10k rows suddenly locks the whole table.

**Detection:** Extended Events (`lock_escalation`) or `sys.dm_tran_locks`.

**Mitigations, in order of preference:**
1. **Batch big DML** — delete/update a few hundred rows at a time in a loop.
2. **Enable `READ_COMMITTED_SNAPSHOT`** — readers stop taking S locks, shrinking the lock count.
3. **Enable optimized locking** (SQL Server 2022+ with ADR + RCSI) — keeps only a transaction-ID lock and releases row/page locks incrementally.
4. **Cover your predicates with indexes** — a scan locks far more rows than a seek.
5. **Last resort:** `ALTER TABLE ... SET (LOCK_ESCALATION = DISABLE)` on that table.

**PostgreSQL has no escalation** — different trade-off; you worry about tuple bloat instead.

Deep dive: [Database Locks](../09-data-access/07-database-locks.md)

</details>

---

### 19. Why is `WITH (NOLOCK)` dangerous beyond just "dirty reads"?

<details>
<summary>Reveal answer</summary>

`NOLOCK` / `READUNCOMMITTED` disables shared locks, so scans can observe the table mid-mutation. It doesn't just allow "reading uncommitted data" — it can return:

- **Duplicate rows** — a row that moves due to a page split during your scan can be read twice.
- **Missing rows** — same mechanism, in reverse.
- **Inconsistent data** — row images from different points in time.

These can happen even when no transaction is rolled back, so retrying doesn't help. Can also surface **error 601**.

The correct modern answer to "we want non-blocking reads" is `ALTER DATABASE ... SET READ_COMMITTED_SNAPSHOT ON`. Readers use row versions instead of locks — same non-blocking behavior, zero correctness risk. A senior interviewer will score you down for defending `NOLOCK` as a performance trick.

Deep dive: [Database Locks](../09-data-access/07-database-locks.md)

</details>

---

### 20. Pessimistic vs optimistic concurrency — how do you implement each?

<details>
<summary>Reveal answer</summary>

**Pessimistic** — lock the row at read time.

```sql
-- SQL Server
BEGIN TRAN;
SELECT Balance FROM Accounts WITH (UPDLOCK, ROWLOCK) WHERE Id = @id;
UPDATE Accounts SET Balance = Balance - @amt WHERE Id = @id;
COMMIT;
```

```sql
-- PostgreSQL
BEGIN;
SELECT balance FROM accounts WHERE id = $1 FOR UPDATE;
UPDATE accounts SET balance = balance - $2 WHERE id = $1;
COMMIT;
```

Best for short critical sections with expected contention and expensive retries.

**Optimistic** — no locks on read; detect conflicts on write via a version column.

```sql
CREATE TABLE Orders (
    Id int PRIMARY KEY,
    Total decimal(10,2),
    RowVersion rowversion NOT NULL
);

UPDATE Orders SET Total = @new
WHERE Id = @id AND RowVersion = @expected; -- @@ROWCOUNT = 0 means conflict
```

```csharp
// EF Core
public class Order
{
    [Timestamp] public byte[] RowVersion { get; set; } = default!;
}
// SaveChangesAsync throws DbUpdateConcurrencyException on conflict
```

Best for web apps with many readers, few conflicts, and easy retries.

**Rule of thumb:** start optimistic; switch to pessimistic for hotspots where conflicts dominate.

Deep dive: [Database Locks](../09-data-access/07-database-locks.md)

</details>

---

### 21. How do you diagnose lock contention and deadlocks in production?

<details>
<summary>Reveal answer</summary>

**SQL Server:**

```sql
-- Current locks and who holds them
SELECT * FROM sys.dm_tran_locks;

-- Active blocking chains
SELECT blocking_session_id, session_id, wait_type, wait_resource
FROM sys.dm_os_waiting_tasks
WHERE blocking_session_id IS NOT NULL;

-- Server-wide top lock waits
SELECT wait_type, waiting_tasks_count, wait_time_ms
FROM sys.dm_os_wait_stats
WHERE wait_type LIKE 'LCK%'
ORDER BY wait_time_ms DESC;
```

Deadlocks (error 1205) are auto-captured in the `system_health` Extended Events session as an XML deadlock graph — pull it with `sys.dm_xe_session_targets`. Always **retry with backoff** in app code; deadlocks are transient.

**PostgreSQL:**

```sql
SELECT pid, relation::regclass, mode, granted FROM pg_locks;

-- Blocking tree
SELECT blocked.pid, blocking.pid, blocked.query, blocking.query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

Deadlocks in Postgres raise `SQLSTATE 40P01`; serialization failures under `SERIALIZABLE` raise `40001`. Both need retries.

Deep dive: [Database Locks](../09-data-access/07-database-locks.md)

</details>

---

### 22. How do you prevent lost updates in a concurrent system?

<details>
<summary>Reveal answer</summary>

Higher isolation levels reduce lost updates but don't eliminate them across transactions. Two proven strategies:

- **Optimistic concurrency** (preferred): add a `RowVersion`/`timestamp` column; on update, include it in the `WHERE`. If 0 rows are affected, the row changed under you — retry or surface a conflict. EF Core throws `DbUpdateConcurrencyException`.
- **Pessimistic locking**: lock the row when reading (`SELECT ... WITH (UPDLOCK, ROWLOCK)`). Safe but causes blocking and is the biggest source of deadlocks — use sparingly.

Optimistic concurrency scales better in web apps; pessimistic is reserved for short critical sections where conflicts are expected and retries are expensive.

Deep dive: [Transaction Isolation Levels](../09-data-access/06-transaction-isolation-levels.md)

</details>

---
