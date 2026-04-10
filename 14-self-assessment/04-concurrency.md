# Concurrency

> Read the questions, think about your answer, then click to reveal.

---

### 1. What is the difference between a Task and a Thread?

<details>
<summary>Reveal answer</summary>

| Aspect | `Thread` | `Task` |
|--------|----------|--------|
| Abstraction level | OS-level thread | Higher-level (work item on thread pool) |
| Creation cost | Expensive (~1MB stack per thread) | Cheap (reuses thread pool threads) |
| Return value | None | `Task<T>` can return a result |
| Exception handling | Crashes the thread unless caught | Propagated via `await` or `.Exception` |
| Cancellation | Manual (`Abort` — deprecated) | `CancellationToken` |
| Continuation | Manual | `await`, `ContinueWith` |

**Rule**: use `Task` and `async/await` for almost everything. Use raw `Thread` only for long-running background work where you need full control.

Deep dive: [Task, Async/Await](../04-concurrency-and-parallelism/03-task-async-await.md)

</details>

---

### 2. What actually happens when you `await` a Task?

<details>
<summary>Reveal answer</summary>

1. The compiler transforms the `async` method into a **state machine** struct.
2. The method runs synchronously until it hits `await` on an incomplete task.
3. The state machine **captures the current state** (locals, position) and **registers a continuation**.
4. The thread is **released** — it returns to the thread pool (or UI message loop).
5. When the awaited task completes, the continuation is **scheduled** on the captured `SynchronizationContext` (UI thread) or the thread pool (if `ConfigureAwait(false)` or no context).
6. Execution resumes at the point after `await`.

**Key insight**: `await` does NOT create a new thread. It frees the current thread and resumes later, which is why it scales for I/O-bound work.

Deep dive: [Task, Async/Await](../04-concurrency-and-parallelism/03-task-async-await.md)

</details>

---

### 3. What does ConfigureAwait(false) do and when should you use it?

<details>
<summary>Reveal answer</summary>

By default, after an `await`, the continuation runs on the original `SynchronizationContext` (e.g., the UI thread in WPF/WinForms). `ConfigureAwait(false)` tells the runtime: "I don't need to resume on the original context — use any available thread pool thread."

**When to use:**
- **Library code**: always use `ConfigureAwait(false)` — you don't know if the caller has a `SynchronizationContext`.
- **ASP.NET Core**: there is no `SynchronizationContext`, so it has no effect — but it doesn't hurt to use it in libraries for portability.
- **UI applications**: do NOT use it if you need to update UI elements after the `await`.

```csharp
// Library code
var data = await httpClient.GetStringAsync(url).ConfigureAwait(false);
// Resumes on thread pool thread — safe for library, avoids deadlock
```

</details>

---

### 4. What is a race condition and how do you prevent it?

<details>
<summary>Reveal answer</summary>

A **race condition** occurs when multiple threads access shared state without synchronization, and the outcome depends on the timing of thread execution.

```csharp
// Classic race condition
private int _count;
void Increment() => _count++; // read-modify-write is NOT atomic
```

**Prevention strategies:**

| Technique | Use case |
|-----------|----------|
| `lock` / `Monitor` | Simple mutual exclusion |
| `SemaphoreSlim` | Limiting concurrent access (e.g., max 3 threads) |
| `Interlocked` | Atomic operations on primitives (`Increment`, `CompareExchange`) |
| `ConcurrentDictionary` | Thread-safe dictionary without manual locking |
| Immutable data | No shared mutable state = no race conditions |
| Channel | Producer-consumer without shared state |

**Best approach**: minimize shared mutable state. Prefer immutable data and message passing.

Deep dive: [Race Conditions](../04-concurrency-and-parallelism/04-race-conditions.md)

</details>

---

### 5. What is a deadlock? How does it happen in async code?

<details>
<summary>Reveal answer</summary>

A **deadlock** occurs when two or more threads are waiting for each other to release a resource, and none can proceed.

**Classic async deadlock** (WPF/WinForms/old ASP.NET):

```csharp
// Controller method
public ActionResult Get()
{
    var result = GetDataAsync().Result; // BLOCKS the SynchronizationContext
    return View(result);
}

async Task<string> GetDataAsync()
{
    var data = await httpClient.GetStringAsync(url);
    // Needs to resume on the SynchronizationContext — but it's blocked!
    return data; // DEADLOCK
}
```

**Prevention:**
1. **Never call `.Result` or `.Wait()`** on async code — use `await` all the way
2. Use `ConfigureAwait(false)` in libraries
3. Acquire locks in a consistent order
4. Use `SemaphoreSlim` with `await WaitAsync()` instead of blocking

Deep dive: [Deadlocks](../04-concurrency-and-parallelism/05-deadlocks.md)

</details>

---

### 6. What is the difference between SemaphoreSlim, lock, and Monitor?

<details>
<summary>Reveal answer</summary>

| Mechanism | Async-friendly | Limits concurrency to N | Scope |
|-----------|---------------|------------------------|-------|
| `lock` (syntactic sugar for `Monitor`) | No | 1 thread | Same process |
| `Monitor` | No | 1 thread (+ `Wait`/`Pulse`) | Same process |
| `SemaphoreSlim` | Yes (`WaitAsync`) | N threads | Same process |
| `Semaphore` | No | N threads | Cross-process |

```csharp
// lock — simple, one thread at a time
lock (_lockObj) { /* critical section */ }

// SemaphoreSlim — async-compatible, limit to N
private readonly SemaphoreSlim _sem = new(3); // max 3 concurrent
await _sem.WaitAsync(cancellationToken);
try { /* work */ }
finally { _sem.Release(); }
```

**Rule**: in async code, always use `SemaphoreSlim` — never `lock`. A `lock` held across an `await` is a compiler error (and a conceptual mistake).

Deep dive: [SemaphoreSlim](../04-concurrency-and-parallelism/06-semaphore-slim.md)

</details>

---

### 7. How does CancellationToken work and why is it important?

<details>
<summary>Reveal answer</summary>

`CancellationToken` enables **cooperative cancellation** — the caller signals cancellation, and the operation checks for it periodically.

```csharp
async Task ProcessAsync(CancellationToken ct)
{
    foreach (var item in items)
    {
        ct.ThrowIfCancellationRequested(); // check at each iteration

        await httpClient.GetAsync(url, ct); // pass to async APIs
    }
}

// Caller
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(30)); // timeout
try
{
    await ProcessAsync(cts.Token);
}
catch (OperationCanceledException)
{
    // Handle cancellation gracefully
}
```

**Why it matters:**
- Prevents wasted work when a request is aborted
- In ASP.NET Core, `HttpContext.RequestAborted` is a `CancellationToken` — pass it through to avoid processing requests the client has abandoned
- Always accept and pass `CancellationToken` in async methods

</details>

---

### 8. When would you use Parallel.ForEach vs Task.WhenAll?

<details>
<summary>Reveal answer</summary>

| Approach | Best for | Threads |
|----------|----------|---------|
| `Parallel.ForEach` | **CPU-bound** work (data parallelism) | Uses thread pool, blocks calling thread |
| `Task.WhenAll` | **I/O-bound** work (concurrent async operations) | No threads blocked |

```csharp
// CPU-bound — image processing
Parallel.ForEach(images, new ParallelOptions { MaxDegreeOfParallelism = 4 },
    img => ProcessImage(img));

// I/O-bound — concurrent HTTP requests
var tasks = urls.Select(url => httpClient.GetAsync(url));
var responses = await Task.WhenAll(tasks);
```

**Common mistake**: using `Parallel.ForEach` with `async` lambdas — it doesn't await them, so fire-and-forget. Use `Task.WhenAll` for async work, or `Parallel.ForEachAsync` (.NET 6+).

Deep dive: [Parallel.ForEach and Invoke](../04-concurrency-and-parallelism/02-parallel-foreach-invoke.md)

</details>

---

### 9. What are Channels and when would you use them?

<details>
<summary>Reveal answer</summary>

`System.Threading.Channels` implement the **producer-consumer** pattern with built-in backpressure and async support.

```csharp
var channel = Channel.CreateBounded<Order>(new BoundedChannelOptions(100)
{
    FullMode = BoundedChannelFullMode.Wait // backpressure
});

// Producer
await channel.Writer.WriteAsync(order);
channel.Writer.Complete(); // signal no more items

// Consumer
await foreach (var item in channel.Reader.ReadAllAsync())
{
    await ProcessOrderAsync(item);
}
```

**Use cases:**
- Background job processing pipelines
- Decoupling producers from consumers (different rates)
- Replacing `BlockingCollection<T>` with async-friendly code
- In-process messaging / event streaming

Channels are lighter than a full message broker (RabbitMQ/Kafka) and ideal for in-process async pipelines.

</details>

---

### 10. What is the difference between concurrency and parallelism?

<details>
<summary>Reveal answer</summary>

- **Concurrency**: dealing with multiple tasks at once (structure). Tasks make progress by interleaving. A single-core machine can be concurrent.
- **Parallelism**: doing multiple tasks at the same time (execution). Requires multiple cores.

| Scenario | Concurrency | Parallelism |
|----------|-------------|-------------|
| Async I/O (web server handling 1000 requests) | Yes | Not necessarily |
| `Parallel.ForEach` on 8 cores | Yes | Yes |
| JavaScript event loop | Yes | No |
| Single-threaded `await` chain | Yes | No |

**Analogy**: Concurrency is one chef switching between multiple dishes. Parallelism is multiple chefs each working on a dish simultaneously.

Deep dive: [Parallelism vs Concurrency](../04-concurrency-and-parallelism/01-parallelism-vs-concurrency.md)

</details>

---

### 11. How would you limit the number of concurrent async operations?

<details>
<summary>Reveal answer</summary>

Several approaches:

```csharp
// 1. SemaphoreSlim
var semaphore = new SemaphoreSlim(10); // max 10 concurrent
var tasks = urls.Select(async url =>
{
    await semaphore.WaitAsync();
    try { return await httpClient.GetAsync(url); }
    finally { semaphore.Release(); }
});
await Task.WhenAll(tasks);

// 2. Parallel.ForEachAsync (.NET 6+) — simplest
await Parallel.ForEachAsync(urls,
    new ParallelOptions { MaxDegreeOfParallelism = 10 },
    async (url, ct) => await httpClient.GetAsync(url, ct));

// 3. Channel with N consumers
// Create N consumer tasks reading from the same channel
```

`Parallel.ForEachAsync` is the cleanest option in .NET 6+. Use `SemaphoreSlim` when you need more control or are on older frameworks.

</details>

---

[Back to index](README.md)
