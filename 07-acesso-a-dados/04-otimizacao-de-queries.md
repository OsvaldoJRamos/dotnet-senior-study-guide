# SQL Query Optimization

## What makes a SQL query slow?

A combination of modeling, indexing, data volume, and how the query is written:

1. **Missing or incorrect indexes** - columns in WHERE, JOIN, ORDER BY, GROUP BY without proper indexes
2. **Functions on columns** - prevents index usage (non-sargable query)
3. **Poorly planned joins** - too many tables, wrong cardinality, joins on non-indexed columns
4. **High data volume** without pagination or partitioning
5. **Unnecessary SELECT \*** - brings more data than needed
6. **Poorly used subqueries and CTEs** - when they could be more efficient joins
7. **Outdated statistics** - optimizer chooses bad plans
8. **Locks and concurrency** - locks, long transactions, misconfigured isolation
9. **Infrastructure** - slow I/O, insufficient memory for cache

### Practical approach

In production:
1. Look at the **execution plan**
2. Check **time and I/O** metrics
3. Validate whether the query is **using the expected indexes**
4. Adjust query, indexes, or model, **measuring impact before and after**

## Parameter Sniffing (SQL Server)

### What it is

SQL Server uses Parameter Sniffing to optimize parameterized queries:

1. On the **first execution**, it takes the actual parameter values
2. Generates an **execution plan optimized** for those values
3. **Caches** the plan to reuse on subsequent executions

### The problem

If the next parameters are **very different** from the first ones, the cached plan can be **terrible** for the new values.

Classic symptom: query runs fast in SSMS (local SQL) and **hangs** when running from the application.

### Solution: OPTION (RECOMPILE)

```sql
SELECT * FROM Pedidos
WHERE ClienteId = @ClienteId
AND DataCriacao BETWEEN @DataInicio AND @DataFim
OPTION (RECOMPILE)  -- forca novo plano a cada execucao
```

### Trade-off

- **Cost**: ~10-50ms compilation overhead per query
- **Benefit**: avoids queries that used to hang for 30-60 seconds
- **Typical result**: query goes from 45s down to ~300ms

### When NOT to use RECOMPILE on everything

- Simple and fast queries (<10ms) - compilation overhead can double the time
- Consumes extra CPU from SQL Server
- Only necessary when there are real parameter sniffing problems

## Non-sargable queries

A query is **sargable** (Search ARGument ABLE) when the optimizer can use indexes to filter.

```sql
-- NON-SARGABLE (function on column prevents index usage)
WHERE ISNUMERIC(Quantity) = 1
WHERE YEAR(DataCriacao) = 2024
WHERE UPPER(Nome) = 'JOAO'

-- SARGABLE (index can be used)
WHERE Quantity IS NOT NULL AND Quantity > 0
WHERE DataCriacao >= '2024-01-01' AND DataCriacao < '2025-01-01'
WHERE Nome = 'JOAO'  -- com collation case-insensitive
```

### Real-world example

```sql
-- Problematico: ISNUMERIC() impede uso de indice
SELECT SUM(CAST(RemainingQuantity AS decimal))
FROM OrderProductDetails opd WITH (NOLOCK)
WHERE opd.OrderId = os1.Id
  AND opd.Quantity != ''
  AND ISNUMERIC(opd.Quantity) = 1  -- nao-sargavel!

-- Plano bom: processa apenas as rows necessarias
-- Plano ruim: calcula ISNUMERIC() para TODAS as rows
```

**Solution**: use a computed column with an index or fix the data type in the database.

## General optimization tips

1. **Create indexes** on columns used in WHERE, JOIN, ORDER BY - but don't overdo it (too many indexes hurt writes)
2. **Avoid SELECT \*** - select only the necessary columns
3. **Use pagination** (OFFSET/FETCH or keyset pagination)
4. **Update statistics** on the database regularly
5. **Use NOLOCK with caution** - can cause dirty reads
6. **Prefer EXISTS over IN** for subqueries with many results
7. **Avoid cursors** - use set-based operations

---

[← Previous: Databases](03-bancos-de-dados.md) | [Back to index](README.md)
