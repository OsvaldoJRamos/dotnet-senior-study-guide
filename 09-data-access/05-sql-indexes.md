# SQL Indexes

Indexes are the single biggest lever for query performance. Without them, the database has to read every row (table scan); with the right ones, it jumps straight to the data it needs.

## What an index actually is

Most indexes are **B-trees** — balanced trees where leaf pages contain pointers (or the rows themselves) sorted by the indexed columns. Lookup cost is `O(log n)` instead of `O(n)`.

```
           [ M ]
          /     \
      [ F ]     [ S ]
     /  |  \   /  |  \
   A-E  G-L M-R  T-Z ...
```

## Types of indexes (SQL Server terminology, similar in Postgres/MySQL)

### Clustered index

Defines the **physical order** of rows in the table. A table can have **only one**. The leaf pages of a clustered index *are* the table rows.

```sql
CREATE CLUSTERED INDEX IX_Orders_Id ON Orders(Id);
```

- Primary key creates a clustered index by default (unless specified otherwise).
- Great for range queries (`BETWEEN`, `ORDER BY` on the clustered key).
- Choose a clustered key that is **narrow, unique, static, and ever-increasing** (e.g., `bigint IDENTITY`). Wide or random keys (like `NEWID()`) cause page splits and fragmentation.

### Non-clustered index

A separate structure that stores the indexed columns plus a **pointer to the row** (the clustered key, or the RID if the table is a heap).

```sql
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId ON Orders(CustomerId);
```

- A table can have **many** non-clustered indexes.
- Lookup via a non-clustered index that doesn't cover all required columns triggers a **key lookup** (extra I/O to fetch missing columns from the clustered index).

### Covering index (with `INCLUDE`)

A non-clustered index that contains **every column the query needs**, removing the key lookup.

```sql
CREATE NONCLUSTERED INDEX IX_Orders_CustomerId_Covering
ON Orders(CustomerId)
INCLUDE (OrderDate, Total);

-- This query is fully served by the index
SELECT OrderDate, Total FROM Orders WHERE CustomerId = 42;
```

Key columns are used for seeking; `INCLUDE` columns are stored at the leaf level only, avoiding the key lookup without bloating the seek structure.

### Composite (multi-column) index

Indexes multiple columns in a defined order.

```sql
CREATE INDEX IX_Orders_Customer_Date ON Orders(CustomerId, OrderDate);
```

- Column order matters. The index can seek on `CustomerId` alone, or on `CustomerId + OrderDate` together, but **not** on `OrderDate` alone.
- Rule of thumb: most selective equality column first, then range columns.

### Unique index

Enforces uniqueness on one or more columns. Often used to back unique constraints.

```sql
CREATE UNIQUE INDEX IX_Users_Email ON Users(Email);
```

### Filtered index

A non-clustered index with a `WHERE` clause. Smaller and cheaper to maintain.

```sql
CREATE INDEX IX_Orders_Pending
ON Orders(CreatedAt)
WHERE Status = 'Pending';
```

Ideal when a small, hot subset of the table is queried often (e.g., unprocessed orders, active users).

### Full-text index

Specialized index for searching words in text columns (`CONTAINS`, `FREETEXT`). Different from `LIKE '%word%'`, which cannot use a regular B-tree index.

### Columnstore index

Stores data **column-by-column**, heavily compressed. Designed for analytic workloads (aggregations over large tables).

- **Clustered columnstore** → fact tables in a data warehouse.
- **Non-clustered columnstore** → analytics on an otherwise OLTP table.

## Index Seek vs Index Scan vs Table Scan

When you read an execution plan, you will see these three operators. Understanding them is essential:

| Operator | What it does | Cost | When you see it |
|---|---|---|---|
| **Index Seek** | Navigates the B-tree to fetch only matching rows | Low | Sargable predicate on an indexed column |
| **Index Scan** | Reads every leaf page of the index | Medium | Predicate not usable by the index, or optimizer estimates lots of rows |
| **Table Scan** (heap) / **Clustered Index Scan** | Reads every row of the table | High | No usable index at all |

### Why a scan happens even when an index exists

- Predicate is **non-sargable** (e.g., `WHERE YEAR(CreatedAt) = 2026`).
- Statistics are stale and the optimizer overestimates row count.
- `SELECT *` forces a key lookup per row — the optimizer may decide a full scan is cheaper.
- Leading column of a composite index is not in the `WHERE` clause.
- Implicit type conversion on the indexed column (e.g., `varchar` column compared to an `nvarchar` parameter).

### Reading the plan

```
SELECT OrderDate FROM Orders WHERE CustomerId = 42;
-- Index Seek on IX_Orders_CustomerId_Covering  ✅ fastest

SELECT * FROM Orders WHERE CustomerId = 42;
-- Index Seek + Key Lookup                      ⚠ extra I/O per row

SELECT * FROM Orders WHERE YEAR(OrderDate) = 2026;
-- Clustered Index Scan                         ❌ non-sargable
```

## The trade-off — indexes are not free

Every index:

- Takes up **disk space**.
- Slows down `INSERT`, `UPDATE`, and `DELETE` (every write must update every affected index).
- Needs **maintenance** — fragmentation grows over time; rebuild or reorganize periodically.
- Consumes **memory** in the buffer pool.

> Rule of thumb: index columns that appear frequently in `WHERE`, `JOIN`, `ORDER BY`, or `GROUP BY`. Avoid indexing low-selectivity columns (e.g., boolean flags). Remove unused indexes — check `sys.dm_db_index_usage_stats`.

## When NOT to index

- Very small tables — a scan is cheap and the optimizer may ignore the index anyway.
- Columns that are almost never filtered on.
- Columns with very low cardinality (few distinct values), unless combined with others in a composite or filtered index.
- Write-heavy tables where read performance is secondary.

## Detecting missing or bad indexes (SQL Server)

```sql
-- Missing index suggestions
SELECT * FROM sys.dm_db_missing_index_details;

-- Index usage — low seeks, high updates = candidate for removal
SELECT * FROM sys.dm_db_index_usage_stats;

-- Fragmentation
SELECT * FROM sys.dm_db_index_physical_stats(DB_ID(), NULL, NULL, NULL, 'LIMITED');
```

Treat "missing index" suggestions as **hints**, not orders — they are based on the current workload snapshot and can recommend overlapping indexes.

---

[← Previous: Query Optimization](04-query-optimization.md) | [Back to index](README.md) | [Next: Transaction Isolation Levels →](06-transaction-isolation-levels.md)
