# Parallelism vs Concurrency

## Difference

- **Concurrency:** multiple tasks *in progress* at the same time (can be interleaved on a single CPU)
- **Parallelism:** multiple tasks *executed literally at the same time* (requires multiple cores)

## When NOT to use parallelism

- **For I/O-bound tasks** (such as API calls or database queries): prefer `async/await`
- **For tasks that are too short**: the overhead of parallelism can make everything slower
- **In code with many accesses to shared resources** (such as lists): parallelism can cause concurrency problems (race conditions)

## How does C# decide how many threads to use?

`Parallel` and `Task` use the **ThreadPool**, which is managed automatically.

You can limit it with `ParallelOptions`:

```csharp
var options = new ParallelOptions { MaxDegreeOfParallelism = 4 };
Parallel.ForEach(lista, options, item => Processar(item));
```

---

[Back to index](README.md) | [Next: Parallel.ForEach and Invoke →](02-parallel-foreach-invoke.md)
