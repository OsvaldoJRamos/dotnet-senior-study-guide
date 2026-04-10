# Parallel.ForEach and Parallel.Invoke

## 1. Parallel.ForEach

Executes loop iterations in parallel.

```csharp
var parallelOptions = new ParallelOptions();
parallelOptions.MaxDegreeOfParallelism = 8;

var stopWatch = new Stopwatch();
stopWatch.Start();

var cepList = new List<CepModel>();

Parallel.ForEach(ceps, parallelOptions, cep =>
{
    cepList.Add(new ViaCepService().GetCep(cep));
});

stopWatch.Stop();
Console.WriteLine($"Elapsed Time {stopWatch.ElapsedMilliseconds} ms");

cepList.ToList().ForEach(cep => Console.WriteLine(cep));
Console.ReadKey();
```

> **Warning:** `List<T>` is not thread-safe. In the example above, using `ConcurrentBag<T>` would be safer.

## 2. Parallel.Invoke

Executes multiple methods simultaneously:

```csharp
Parallel.Invoke(
    () => Metodo1(),
    () => Metodo2(),
    () => Metodo3()
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
