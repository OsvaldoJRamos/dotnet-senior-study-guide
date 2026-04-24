# Parallelism vs Concurrency

## Difference

- **Concurrency:** multiple tasks *in progress* at the same time (can be interleaved on a single CPU)
- **Parallelism:** multiple tasks *executed literally at the same time* (requires multiple cores)

## I/O-bound vs CPU-bound — pick the tool from the workload

Every operation falls into one of these, and the right tool depends on which.

| | I/O-bound | CPU-bound |
|---|---|---|
| What the thread does during the work | Waits | Executes |
| Bottleneck | Network / disk / external service latency | Number of CPU cores |
| Examples | HTTP, DB, queue, file read | Hash, compression, parsing, math |
| Right tool | `async`/`await`, `Task.WhenAll`, `Parallel.ForEachAsync` | `Parallel.For`/`ForEach`, PLINQ, `Task.Run` |

> Analogy: an I/O-bound operation is a phone call — the worker doesn't have to sit there holding the receiver, they can do something else and pick up when it rings. A CPU-bound operation is filling in a spreadsheet by hand — the only way to go faster is more workers.

Mixing the two is the most common concurrency mistake: `Parallel.ForEach` over HTTP calls just blocks pool threads on network waits; `Task.Run` sprayed over a tight CPU loop shuffles work between threads without adding cores.

## When NOT to use parallelism

- **For I/O-bound tasks** (such as API calls or database queries): prefer `async/await`
- **For tasks that are too short**: the overhead of parallelism can make everything slower
- **In code with many accesses to shared resources** (such as lists): parallelism can cause concurrency problems (race conditions)

## How does C# decide how many threads to use?

`Parallel` and `Task` use the **ThreadPool**, which is managed automatically.

You can limit it with `ParallelOptions`:

```csharp
var options = new ParallelOptions { MaxDegreeOfParallelism = 4 };
Parallel.ForEach(list, options, item => Process(item));
```

---

[Back to index](README.md) | [Next: Parallel.ForEach and Invoke →](02-parallel-foreach-invoke.md)
