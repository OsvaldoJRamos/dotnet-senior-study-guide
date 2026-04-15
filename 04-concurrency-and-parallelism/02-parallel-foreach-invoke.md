# Parallel.ForEach and Parallel.Invoke

## 1. Parallel.ForEach

Executes loop iterations in parallel. Designed for **CPU-bound** work — do **not** use for I/O (HTTP, DB). For parallel I/O, see `Parallel.ForEachAsync` below.

```csharp
var parallelOptions = new ParallelOptions { MaxDegreeOfParallelism = 8 };

var payloads = Enumerable.Range(0, 10_000)
    .Select(_ => RandomNumberGenerator.GetBytes(1024))
    .ToArray();

var hashes = new List<byte[]>(); // BUG: see warning below

Parallel.ForEach(payloads, parallelOptions, payload =>
{
    var hash = SHA256.HashData(payload); // pure CPU work
    hashes.Add(hash);
});
```

> **Warning:** `List<T>.Add` from multiple threads is **not thread-safe**. Concurrent `Add` calls can corrupt the internal array, drop items silently, or throw `IndexOutOfRangeException` / `ArgumentException` when the list resizes mid-write. Use `ConcurrentBag<T>`, `ConcurrentQueue<T>`, or a local aggregation with `Parallel.ForEach`'s `localInit`/`localFinally` overload.

### Parallel.ForEachAsync (.NET 6+) — for I/O

Use this, not `Parallel.ForEach`, when each iteration is an async I/O call:

```csharp
await Parallel.ForEachAsync(
    ceps,
    new ParallelOptions { MaxDegreeOfParallelism = 4 },
    async (cep, ct) => await new ViaCepService().GetCepAsync(cep, ct));
```

## 2. Parallel.Invoke

Executes multiple methods simultaneously:

```csharp
Parallel.Invoke(
    () => Method1(),
    () => Method2(),
    () => Method3()
);
```

Useful when you have a fixed number of independent operations to execute in parallel.

## When to use each one

| Scenario | Use |
|---|---|
| Process a collection in parallel | `Parallel.ForEach` |
| Execute N independent methods | `Parallel.Invoke` |
| I/O-bound operations (API, database) | `Task` with `async/await` (not `Parallel`) |

---

[← Previous: Parallelism vs Concurrency](01-parallelism-vs-concurrency.md) | [Back to index](README.md) | [Next: Task, async/await →](03-task-async-await.md)
