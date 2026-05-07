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

Deep dive: [Task, Async/Await](../06-concurrency-and-parallelism/03-task-async-await.md)

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

Deep dive: [Task, Async/Await](../06-concurrency-and-parallelism/03-task-async-await.md)

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

Deep dive: [Race Conditions](../06-concurrency-and-parallelism/04-race-conditions.md)

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

Deep dive: [Deadlocks](../06-concurrency-and-parallelism/05-deadlocks.md)

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

Deep dive: [SemaphoreSlim](../06-concurrency-and-parallelism/06-semaphore-slim.md)

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

Deep dive: [Parallel.ForEach and Invoke](../06-concurrency-and-parallelism/02-parallel-foreach-invoke.md)

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

Deep dive: [Parallelism vs Concurrency](../06-concurrency-and-parallelism/01-parallelism-vs-concurrency.md)

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

### 12. What does the C# `lock` statement actually compile to?

<details>
<summary>Reveal answer</summary>

It depends on the static type of the lock target:

- **Any reference type other than `System.Threading.Lock`** — lowers to `Monitor.Enter(obj, ref bool taken)` + `try/finally` with `Monitor.Exit` guarded by `taken`. The `ref bool` pattern ensures we only release if we actually acquired.
- **`System.Threading.Lock` (.NET 9+ / C# 13)** — lowers to `using (target.EnterScope()) { ... }`. `EnterScope` returns a `ref struct` whose `Dispose()` releases the lock. Faster than the Monitor path and clearer in intent.

Key consequence: the same lock target **must** have the static type `System.Threading.Lock` for the optimized path. If you hold one via `object`, the compiler emits the old Monitor path.

Deep dive: [Locks in Depth](../06-concurrency-and-parallelism/07-locks-in-depth.md)

</details>

---

### 13. Why is `await` inside a `lock` a compile error, and what's the underlying reason?

<details>
<summary>Reveal answer</summary>

`await` inside `lock { ... }` raises **CS1996** at compile time — it's not a runtime issue.

The underlying reason is **thread affinity**. Both `Monitor` (what plain `lock` uses) and `System.Threading.Lock` require the **same thread** that acquired the lock to release it. After an `await`, the continuation may resume on a **different thread** — that thread wouldn't be allowed to `Monitor.Exit`, so the lock would leak forever and freeze all subsequent callers.

The async-friendly alternative is `SemaphoreSlim`, which is **not** thread-affine — any thread may call `Release`:

```csharp
await _gate.WaitAsync(ct);
try { await DoWorkAsync(); }
finally { _gate.Release(); }
```

Deep dive: [Locks in Depth](../06-concurrency-and-parallelism/07-locks-in-depth.md), [Deadlocks](../06-concurrency-and-parallelism/05-deadlocks.md)

</details>

---

### 14. When would you pick `ReaderWriterLockSlim` over `lock`? When would `lock` still win?

<details>
<summary>Reveal answer</summary>

Use **`ReaderWriterLockSlim`** when reads vastly outnumber writes **and** each read does non-trivial work, so parallelism among readers pays off. Multiple threads can hold the read lock simultaneously; writers are exclusive.

```csharp
_rw.EnterReadLock();  try { return _cache[key]; }  finally { _rw.ExitReadLock(); }
_rw.EnterWriteLock(); try { _cache[key] = value; } finally { _rw.ExitWriteLock(); }
```

`EnterUpgradeableReadLock` is the important third mode — one upgradeable reader plus many plain readers. Lets you check and then upgrade to a write without the classic read-release-write race.

**Why `lock` often wins anyway:** `ReaderWriterLockSlim` has higher per-call overhead. For short critical sections (e.g., a hashmap lookup), a plain `lock` finishes faster than the bookkeeping for a read lock. Also, `ConcurrentDictionary` is usually the better answer than either. **Benchmark before switching.**

Deep dive: [Locks in Depth](../06-concurrency-and-parallelism/07-locks-in-depth.md)

</details>

---

### 15. What is `System.Threading.Lock` (.NET 9+) and what does it buy you?

<details>
<summary>Reveal answer</summary>

A **dedicated lock type** added in .NET 9 / C# 13, replacing the decades-old idiom `private readonly object _lockObj = new();`.

```csharp
private readonly System.Threading.Lock _gate = new();
lock (_gate) { /* critical section */ }
```

Benefits:

1. **Intent** — the type literally says "this is a lock".
2. **Performance** — when the static type is `Lock`, the compiler lowers `lock` to `using (_gate.EnterScope())`, which bypasses object-header thin-lock machinery and is measurably faster under contention.
3. **Correctness by design** — nobody can accidentally `Monitor.Enter` on it through an `object` reference.
4. **Warning on misuse** — the compiler warns if you cast a known `Lock` to another type and lock it.

Caveat: if you assign a `Lock` into `object` or a generic `T` and lock there, you silently lose the optimized path.

Deep dive: [Locks in Depth](../06-concurrency-and-parallelism/07-locks-in-depth.md)

</details>

---

### 16. When is `Interlocked` the right choice, and where does it stop being enough?

<details>
<summary>Reveal answer</summary>

`Interlocked` provides **atomic operations on a single 32/64-bit or native-sized field** — lock-free, no thread affinity, no blocking. Use it for:

- Counters: `Interlocked.Increment(ref _count);`
- Flags / single-reference swaps: `Interlocked.Exchange(ref _current, next);`
- Lock-free CAS patterns (lazy init, lock-free stacks): `Interlocked.CompareExchange(ref field, newVal, expected);`
- Atomic `long` read on 32-bit CPUs: `Interlocked.Read(ref _big);`

**Where it stops being enough:** anything that requires two fields to update consistently. Composing two `Interlocked` calls is **not** atomic:

```csharp
// Still a race — another thread can decrement between the check and the decrement.
if (Interlocked.Read(ref x) > 0) Interlocked.Decrement(ref x);
```

For compound invariants, you need a real lock or a transactional data structure.

Deep dive: [Locks in Depth](../06-concurrency-and-parallelism/07-locks-in-depth.md)

</details>

---

### 17. Why does `await` release the thread for I/O work but not for pure CPU work?

<details>
<summary>Reveal answer</summary>

`await` frees the calling thread only when the awaited `Task` depends on something **outside a managed thread** to complete.

- **I/O-bound**: the wait is handled by the kernel / network stack / disk controller. Once the operation is in flight, no managed thread has to sit on it, so the state machine registers a continuation and the thread goes back to the pool.
- **CPU-bound without `Task.Run`**: there is no external wait — executing the code *is* the work. Whatever thread entered the method runs it. `await` has nothing to release.
- **CPU-bound wrapped in `Task.Run`**: the work is explicitly moved onto a pool thread, so the caller now has something external to observe and can suspend.

Rule: `await` releases the caller when the work is being done by something *other* than a managed thread (kernel, hardware, another pool thread). Otherwise someone still has to run the code.

Deep dive: [Task, Async/Await](../06-concurrency-and-parallelism/03-task-async-await.md)

</details>

---

### 18. Why should `Task.Run` stay in the caller instead of being hidden inside a library method?

<details>
<summary>Reveal answer</summary>

`Task.Run` is a decision about **which thread runs the work**. A library has no business making that call for its consumer:

- The caller may already be on a background thread — hiding `Task.Run` inside doubles the hop.
- The caller may need UI-thread affinity for what follows — a library hop silently breaks it.
- The work may be cheap enough that the thread switch costs more than it saves.

Library contract: expose synchronous work as synchronous and genuine async work as async. Let the caller decide when to offload with `Task.Run`. This is also why a method that just wraps `Task.Run(() => Sync())` and calls itself async is called **fake async** — it fools the caller into thinking the method is scalable when it still costs one thread per call.

Deep dive: [Task, Async/Await](../06-concurrency-and-parallelism/03-task-async-await.md)

</details>

---

### 19. What makes fire-and-forget dangerous, and how do you make it safe?

<details>
<summary>Reveal answer</summary>

An unawaited `Task` is fire-and-forget — nobody is observing it. Three problems:

1. **Lost exceptions.** A faulted task's unobserved exception only surfaces via `TaskScheduler.UnobservedTaskException` when the runtime is about to trigger exception escalation policy — the timing is delayed and unpredictable, and the event is easy to miss.
2. **Shutdown can cut it mid-flight.** The host doesn't know a background task is still running.
3. **`async void` is worse.** Exceptions rethrow on the captured `SynchronizationContext`, bypassing the caller's `try`/`catch` and potentially crashing a UI or classic ASP.NET app.

Safe patterns:

```csharp
// Minimum viable: catch and log.
_ = Task.Run(async () =>
{
    try { await DoWorkAsync(); }
    catch (Exception ex) { _logger.LogError(ex, "Background work failed"); }
});
```

Better: hand the work to a `Channel<T>` or a hosted `BackgroundService` that owns lifecycle, cancellation, and error handling. `Task.Run` with a catch is a bandage, not a design.

Deep dive: [Task, Async/Await](../06-concurrency-and-parallelism/03-task-async-await.md)

</details>

---

### 20. The .NET thread pool tracks two separate categories of threads. What are they, and why does that distinction matter?

<details>
<summary>Reveal answer</summary>

Worker threads and I/O completion threads are tracked independently — `ThreadPool.GetMinThreads`/`GetMaxThreads` return both numbers.

| Category | Used for |
|---|---|
| Worker threads | CPU-bound work: `Task.Run`, `QueueUserWorkItem`, `Timer` callbacks, registered waits |
| I/O completion threads | Completions from async I/O bound to the pool (Windows IOCP) |

Why it matters: starving one category does not necessarily starve the other. A loop that hammers `Task.Run` with `.Result` blocks **worker** threads; if your monitoring only watches IOCP threads, the bug is invisible. When diagnosing thread pool issues, always check both — `GetAvailableThreads`, the `threadpool-queue-length` EventCounter, and the `ThreadCount` property all give signals per category.

Deep dive: [The Thread Pool](../06-concurrency-and-parallelism/08-thread-pool.md)

</details>

---

### 21. What is "thread pool starvation" and why does `SetMinThreads` not fix it properly?

<details>
<summary>Reveal answer</summary>

**Starvation** is when so many pool threads are blocked that throughput collapses and the pool can't grow fast enough to recover. The classic trigger is synchronous waits on async work:

```csharp
var data = httpClient.GetStringAsync(url).Result;   // blocks a pool thread
```

Each `.Result` parks one worker thread sitting on a network read. Multiply by N concurrent requests → N stuck threads → empty pool. Hill climbing tries to grow the pool, but new threads are added at a **rate-limited** pace, so latency spikes for every request until the pool catches up — sometimes 30+ seconds.

`SetMinThreads(n, n)` raises the floor below which threads are created instantly — that masks the symptom for the first `n` blocked calls. But:

- It doesn't fix the actual cause (blocking).
- The next burst still hits the rate-limited path.
- It can degrade steady-state throughput because hill climbing is overridden.

The MS Learn `SetMinThreads` page explicitly says: *"In most cases, the thread pool will perform better with its own algorithm for allocating threads."*

The real fix is being `async` end-to-end so threads are released during I/O and never sit on `.Result`/`.Wait()`.

Deep dive: [The Thread Pool](../06-concurrency-and-parallelism/08-thread-pool.md)

</details>

---

### 22. The thread pool uses local queues, a global queue, and work stealing. What is each used for?

<details>
<summary>Reveal answer</summary>

- **Global queue** — receives work submitted from outside the pool (e.g., `Task.Run` called from a non-pool thread, an HTTP request handler scheduling a task).
- **Per-thread local queue** — receives work submitted from inside the pool. When a pool thread does `Task.Run(...)`, the new task lands on **its own** local queue. Local queues use LIFO ordering, which exploits cache locality (the just-created task often touches hot data).
- **Work stealing** — when a thread's local queue is empty, it checks the global queue, then steals from another thread's local queue.

Confirmation from MS Learn (`SetMinThreads` remarks, on the cost of over-provisioning): *"Worker threads may take more CPU time in dequeuing work items due to having to scan more threads to **steal work** from."*

Practical consequence: `Task.Run` from inside the pool is cheap (no global contention); `Task.Run` from outside hits the global queue and is slightly more expensive.

Deep dive: [The Thread Pool](../06-concurrency-and-parallelism/08-thread-pool.md)

</details>

---

### 23. What is `BoundedChannelFullMode` and why does the default value give you backpressure?

<details>
<summary>Reveal answer</summary>

`BoundedChannelFullMode` controls what happens when a producer writes to a full bounded channel. The four modes (from MS Learn):

| Mode | Behavior |
|---|---|
| `Wait` (default) | Waits for space to be available before completing the write. |
| `DropNewest` | Removes and ignores the **newest** queued item to make room. |
| `DropOldest` | Removes and ignores the **oldest** queued item to make room. |
| `DropWrite` | Drops the **item being written**. |

`Wait` is what gives you **backpressure**: the producer's `WriteAsync` simply doesn't complete until the consumer has freed a slot. The producer naturally throttles to the consumer's pace, and queue size is bounded by capacity.

Without backpressure (unbounded channel, or a `Drop*` mode), a slow consumer either blows up memory (unbounded) or silently loses data (drop modes). `Drop*` modes are appropriate for telemetry/metrics where freshness beats completeness. For business data (orders, payments), `Wait` is the only safe choice.

```csharp
var channel = Channel.CreateBounded<Order>(new BoundedChannelOptions(1000));
// FullMode defaults to Wait — Wait is the right default for almost everything
```

Deep dive: [Channels](../06-concurrency-and-parallelism/09-channels.md)

</details>

---

### 24. What do `SingleReader` and `SingleWriter` do on a `Channel`?

<details>
<summary>Reveal answer</summary>

They are **promises** the caller makes about the access pattern, not guards the runtime enforces. When set, the channel implementation switches to a specialized fast path with fewer locks (and in the single-reader case, fewer interlocked operations).

From the official Stephen Toub article: *"When `SingleReader` is true, the implementation not only avoids locks when reading, it also avoids interlocked operations when reading, significantly reducing the overheads involved in consuming from the channel."*

Set them whenever the access pattern matches — most pipelines have one consumer and a known set of producers. Lying about it (e.g., setting `SingleReader = true` and reading from two threads) is undefined behavior and can corrupt internal state.

```csharp
var channel = Channel.CreateBounded<Work>(new BoundedChannelOptions(100)
{
    SingleReader = true,   // I promise: at most one ReadAsync at a time
    SingleWriter = false   // multiple producers
});
```

Deep dive: [Channels](../06-concurrency-and-parallelism/09-channels.md)

</details>

---

### 25. What is the `GetOrAdd` factory pitfall in `ConcurrentDictionary`, and how do you fix it?

<details>
<summary>Reveal answer</summary>

The factory delegate passed to `GetOrAdd` (and `AddOrUpdate`) **runs outside the dictionary's locks**, which means **two threads racing on the same key can both run the factory**. Only one of the produced values wins the slot; the other is silently discarded.

The MS Learn remarks state this directly: *"delegates for these methods are called outside the locks to avoid the problems that can arise from executing unknown code under a lock. Therefore, the code executed by these delegates is not subject to the atomicity of the operation."*

For factories with side effects (logging, DB calls, file I/O) this is a real bug — the side effect runs N times, even though only one cache entry remains.

Fix with `Lazy<T>`:

```csharp
private readonly ConcurrentDictionary<string, Lazy<Config>> _cache = new();

public Config Get(string key) =>
    _cache.GetOrAdd(key, k => new Lazy<Config>(() => LoadFromDb(k))).Value;
```

Multiple threads may build a `Lazy<Config>`, but only one wins the dictionary slot, and `Lazy<T>` ensures `LoadFromDb` runs at most once.

Deep dive: [Concurrent Collections](../06-concurrency-and-parallelism/10-concurrent-collections.md)

</details>

---

### 26. When is `ConcurrentBag<T>` the right pick, and when is it the *wrong* one?

<details>
<summary>Reveal answer</summary>

The official remarks pin it down: *"`ConcurrentBag<T>` is a thread-safe bag implementation, optimized for scenarios where the same thread will be both producing and consuming data stored in the bag."*

Each thread has its own local list. `Add` writes locally (zero contention), `TryTake` reads locally first and only **steals** from another thread's list when local is empty.

**Right fit:** embarrassingly parallel work where each thread accumulates results, possibly drains its own results, with rare cross-thread takes. Example: `Parallel.ForEach` workers collecting partial sums or hashes into per-thread buckets.

**Wrong fit:** classic producer/consumer with **distinct** producer threads and consumer threads. Every consume becomes a steal, which is more expensive than the dedicated FIFO path in `ConcurrentQueue` or the wait-aware path in `Channel<T>`. For that shape, use `ConcurrentQueue<T>` or `Channel<T>`.

Deep dive: [Concurrent Collections](../06-concurrency-and-parallelism/10-concurrent-collections.md)

</details>

---

### 27. You need an in-process producer/consumer pipeline. Should you use `BlockingCollection<T>` or `Channel<T>`?

<details>
<summary>Reveal answer</summary>

Almost always `Channel<T>` in modern code.

| | `BlockingCollection<T>` | `Channel<T>` |
|---|---|---|
| Wait semantics | Synchronous (`Take()` blocks the thread) | Async (`ReadAsync` returns `ValueTask<T>`) |
| Thread cost while waiting | One full OS thread parked | Zero — thread released to pool |
| Backpressure (bounded) | Yes (`Add` blocks) | Yes (`WriteAsync` async-waits) |
| `IAsyncEnumerable<T>` consumer | No | Yes (`ReadAllAsync`) |
| Allocation profile | Heap-y | `ValueTask`-based fast paths |
| `Single*` optimizations | No | `SingleReader`/`SingleWriter` |

`BlockingCollection<T>` still earns its keep in **synchronous-only** code (legacy console tools, simple worker loops without `async`). For anything that already uses `await`, `Channel<T>` is the right default — same producer/consumer shape, async-first.

Deep dive: [Channels](../06-concurrency-and-parallelism/09-channels.md), [Concurrent Collections](../06-concurrency-and-parallelism/10-concurrent-collections.md)

</details>

---

### 28. How many `TaskStatus` values are there, and which are terminal?

<details>
<summary>Reveal answer</summary>

The `TaskStatus` enum has **exactly 8 values** (per MS Learn):

| # | Name | Terminal? |
|---|---|---|
| 0 | `Created` | no |
| 1 | `WaitingForActivation` | no |
| 2 | `WaitingToRun` | no |
| 3 | `Running` | no |
| 4 | `WaitingForChildrenToComplete` | no |
| 5 | `RanToCompletion` | **yes** |
| 6 | `Canceled` | **yes** |
| 7 | `Faulted` | **yes** |

Three terminal states. Once a task lands on one, it never leaves.

Most async-method tasks are born in `WaitingForActivation`, not `Created`. `Task.Run` produces tasks that go `WaitingToRun → Running → ...`. `Task.FromResult` produces tasks that are born already in `RanToCompletion`.

Deep dive: [Task Lifecycle](../06-concurrency-and-parallelism/11-task-lifecycle.md)

</details>

---

### 29. What's the difference between `Task.IsCompleted` and `Task.IsCompletedSuccessfully`?

<details>
<summary>Reveal answer</summary>

`IsCompleted` is `true` for **any** terminal state — `RanToCompletion`, `Faulted`, **or** `Canceled`. It's a "did this task end?" check, not a "did this task succeed?" check.

`IsCompletedSuccessfully` (introduced in .NET Core 2.0 / .NET Standard 2.1) is `true` only for `RanToCompletion` — per the docs, it "Gets whether the task ran to completion."

The classic bug:

```csharp
// BUG — task may be Faulted or Canceled; .Result rethrows
if (task.IsCompleted)
    UseResult(task.Result);

// CORRECT
if (task.IsCompletedSuccessfully)
    UseResult(task.Result);
```

Use `IsCompletedSuccessfully` whenever you want to peek without risk of throwing.

Deep dive: [Task Lifecycle](../06-concurrency-and-parallelism/11-task-lifecycle.md)

</details>

---

### 30. How do `await` and `.Wait()` / `.Result` differ in how they surface exceptions?

<details>
<summary>Reveal answer</summary>

`await` rethrows the **original** exception unwrapped — `InvalidOperationException` is `InvalidOperationException`. Cancellation surfaces as `TaskCanceledException` (a subclass of `OperationCanceledException`).

`.Wait()` and `.Result` always wrap in `AggregateException`. The official `Task.Wait` docs are explicit: *"`AggregateException` ... The `InnerExceptions` collection contains information about the exception or exceptions."* Even cancellation: a `Wait()` on a canceled task throws `AggregateException` wrapping `TaskCanceledException` — you can't `catch (OperationCanceledException)` directly.

If you absolutely must block, `task.GetAwaiter().GetResult()` is the lesser evil: same blocking cost, but exceptions come through unwrapped (it's what `await` itself uses internally). Starvation and deadlock risks remain — async-all-the-way is still the right answer when possible.

```csharp
// await — natural exception type
try { await DoAsync(); }
catch (InvalidOperationException) { /* clean */ }

// .Result — wraps everything
try { var v = DoAsync().Result; }
catch (AggregateException ae) { /* drill into ae.InnerException */ }

// GetAwaiter().GetResult() — blocks but unwraps
try { var v = DoAsync().GetAwaiter().GetResult(); }
catch (InvalidOperationException) { /* unwrapped, like await */ }
```

Deep dive: [Task, Async/Await](../06-concurrency-and-parallelism/03-task-async-await.md), [Task Lifecycle](../06-concurrency-and-parallelism/11-task-lifecycle.md)

</details>

---

### 31. Three conditions must hold for a cancelled task to land in `Canceled` (not `Faulted`). What are they?

<details>
<summary>Reveal answer</summary>

From the official Task Cancellation guide: *"When a task instance observes an `OperationCanceledException` thrown by the user code, it compares the exception's token to its associated token... If the tokens are same and the token's `IsCancellationRequested` property returns `true`, the task interprets this as acknowledging cancellation and transitions to the `Canceled` state."*

Three conditions, all must hold:

1. An `OperationCanceledException` is thrown from the task body.
2. The exception's `CancellationToken` **equals** the token passed to the task-creating API (e.g., `Task.Run(..., ct)`).
3. That token's `IsCancellationRequested` is `true`.

Miss any of the three → task ends up `Faulted`.

Canonical safe pattern:

```csharp
var task = Task.Run(() =>
{
    ct.ThrowIfCancellationRequested();   // throws OCE(ct) if signalled
    DoSomeWork(ct);
}, ct);   // ← same token to Task.Run
```

`ct.ThrowIfCancellationRequested()` constructs the OCE with the right token automatically. Throwing `new OperationCanceledException()` without a token, or passing a different token to `Task.Run`, lands in `Faulted`.

> Returning normally from a delegate — even after polling `ct.IsCancellationRequested` — transitions to `RanToCompletion`, **not** `Canceled`. Only the throw-with-matching-token pattern produces `Canceled`.

Deep dive: [Task Lifecycle](../06-concurrency-and-parallelism/11-task-lifecycle.md)

</details>

---

### 32. When is `Task.FromResult` legitimate, and when is it the "fake-async" anti-pattern?

<details>
<summary>Reveal answer</summary>

`Task.FromResult<T>(value)` returns a `Task<T>` already in `RanToCompletion`. It's a way to satisfy a `Task<T>`-returning signature with an instantly-known value.

**Legitimate uses:**

1. **Async interface, sync fast path** — cache hit returns `Task.FromResult(cached)`; cache miss does real async I/O.
2. **Implementing `Task`-returning interfaces from sync code** — use `Task.CompletedTask` (the void equivalent).
3. **Mocks / test stubs** that satisfy an async signature.

**Anti-pattern (fake async):**

```csharp
public Task<int> SumAsync(int[] nums) => Task.FromResult(nums.Sum());
```

The signature claims "this might suspend; await me." It lies — `Sum()` is instant and synchronous. The caller pays state-machine + continuation costs and gains nothing. When this becomes a convention ("everything is async for uniformity"), real async I/O becomes indistinguishable from facade.

For hot synchronous fast paths, prefer `ValueTask<T>` — a struct that wraps the synchronous value with no heap allocation.

Note (.NET 6+): per the docs, *"for some `TResult` types and some result values, this method may return a cached singleton object rather than allocating a new object."* So `Task.FromResult(true)` and similar may be allocation-free. Doesn't change the semantic: don't fake async.

Deep dive: [Task, Async/Await](../06-concurrency-and-parallelism/03-task-async-await.md), [Async and Memory](../05-memory-and-performance/06-async-and-memory.md)

</details>

---

### 33. When should code stay synchronous instead of going async?

<details>
<summary>Reveal answer</summary>

The right question is *"what do I gain by freeing this thread?"* — not *"can this be async?"*. Async has costs (state machine allocation, continuation scheduling, uglier stack traces, `ExecutionContext` capture, viral propagation up the call stack). When the answer to that question is "nothing", async is overhead.

Sync wins by default when:

- **Pure CPU work** with no I/O — there's nothing to wait on.
- **Single-shot CLI tools, scripts, migrations, startup code** — no other users to serve while you wait.
- **Naturally synchronous library code** (parsers, validators, formatters) — `JsonSerializer.Deserialize(string)` is sync, and that's correct.
- **Hot paths where state-machine cost outweighs the work** (consider `ValueTask` first, then sync if even that's too heavy).

Async wins when **someone could use this thread while you wait**:

- Web servers (always another request to serve).
- UI apps with I/O (blocking the UI thread freezes the app).
- Batch processes with many concurrent I/O operations.

Heuristic: **sync by default if no I/O, or if single-shot. Async if there's I/O *and* concurrency (server, UI, parallel batch).**

Deep dive: [Task, Async/Await](../06-concurrency-and-parallelism/03-task-async-await.md)

</details>

---

### 34. A request reads from a database, runs a heavy transform, then writes to storage. How do you structure the `await`s?

<details>
<summary>Reveal answer</summary>

Pick the tool per workload, not one tool for everything:

```csharp
public async Task<Report> GenerateReportAsync(int userId, CancellationToken ct)
{
    var data      = await _db.GetUserDataAsync(userId, ct);         // I/O — thread released
    var processed = await Task.Run(() => HeavyTransform(data), ct); // CPU — runs on a pool thread
    await _storage.SaveAsync(processed, ct);                        // I/O — thread released
    return new Report(processed);
}
```

Three `await`s, three distinct reasons to suspend:

- The DB and storage calls are **I/O**: `await` delegates the wait to the OS and returns the thread to the pool.
- The transform is **CPU**: `Task.Run` is needed to keep the calling thread responsive. Without it, the method would run the transform synchronously on the caller, even though the signature says `async`.

If `HeavyTransform` were cheap, you'd drop `Task.Run` entirely — the thread switch would cost more than the work.

Deep dive: [Task, Async/Await](../06-concurrency-and-parallelism/03-task-async-await.md)

</details>

---

[Back to index](README.md)
