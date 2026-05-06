# Task, async/await and PLINQ

## 3. Task with async/await

Ideal for **asynchronous concurrency**, but can also be used for parallelism with `Task.WhenAll`:

```csharp
var tasks = new[]
{
    Task.Run(() => Calculate(1)),
    Task.Run(() => Calculate(2)),
    Task.Run(() => Calculate(3))
};

await Task.WhenAll(tasks);
```

### Practical example with I/O:

```csharp
var cepTaskList = ceps.Select(cep => new ViaCepService().GetCepAsync(cep));
var cepList = await Task.WhenAll(cepTaskList);
```

## Why `await` frees a thread for I/O but not pure CPU

`await` does not magically suspend anything. It frees the calling thread only when the awaited `Task` depends on something **outside a managed thread** to complete.

- **I/O-bound work** — the wait is handled by the kernel / network stack / disk controller. Once the operation is in flight, no managed thread needs to sit on it. The state machine registers a continuation and the thread returns to the pool. When the OS signals completion, some thread (not necessarily the original) picks up the continuation.
- **CPU-bound work without `Task.Run`** — there is no external wait: executing the code *is* the work. Whatever thread called the method is the thread that runs it. `await` has nothing to release.
- **CPU-bound work inside `Task.Run`** — the work is explicitly moved onto a pool thread, so the caller now has something external to observe and `await` can release it.

One-line rule: `await` frees the caller when the work is done by something other than a managed thread (kernel, hardware, or a different pool thread). Otherwise someone still has to run the code.

> The async docs state it directly: "The `async` and `await` keywords don't cause extra threads to be created. Async methods don't require multithreading because an async method doesn't run on its own thread." Concurrency comes from the wait being delegated — not from spinning up threads.

## 4. PLINQ (Parallel LINQ)

Allows parallelizing LINQ queries:

```csharp
var result = list
    .AsParallel()
    .Where(x => Process(x))
    .ToList();
```

## Task.Run vs async/await — when to use each one

| Scenario | Use |
|---|---|
| Heavy CPU-bound work | `Task.Run` |
| I/O calls (API, database, file) | `async/await` directly (without `Task.Run`) |
| Multiple simultaneous I/O operations | `Task.WhenAll` |
| Heavy query on a large collection | `PLINQ` |

### Library rule: keep `Task.Run` in the caller

Don't hide `Task.Run` inside library methods. It takes the decision away from the consumer:

- If the caller is already on a background thread, you double the hop for nothing.
- If the caller needed UI-thread affinity for what follows, you silently break it.
- If the work turns out to be cheap, you paid for a thread switch with no gain.

Expose synchronous work as synchronous and genuine async work as async. Let the caller decide when to offload with `Task.Run`.

## Common mistakes

### Do not use .Result or .Wait()

```csharp
// WRONG - can cause deadlock on classic ASP.NET / WinForms / WPF
var result = MyOperationAsync().Result;

// CORRECT - use await
var result = await MyOperationAsync();
```

The deadlock happens **only when a `SynchronizationContext` is captured** — i.e., classic ASP.NET (System.Web), WinForms, WPF. It does **not** happen in ASP.NET Core or console apps (neither has a sync context by default).

**Mechanism:** inside `MyOperationAsync`, an `await` captures the current sync context. When the awaited operation completes, the continuation is posted back to that context to resume. But the caller is blocking that same context with `.Result` / `.Wait()` — so the continuation never runs, and the caller waits forever.

Even outside the deadlock case (e.g., ASP.NET Core), `.Result`/`.Wait()` parks a pool thread doing nothing — the canonical thread pool starvation pattern. See [The Thread Pool](08-thread-pool.md).

#### `.Wait()` / `.Result` wraps exceptions in `AggregateException`

`await` rethrows the original exception unwrapped. The blocking calls don't:

```csharp
// await — catches InvalidOperationException directly
try { await DoAsync(); }
catch (InvalidOperationException) { /* clean */ }

// .Result — catches AggregateException; the real one is in InnerException
try { var v = DoAsync().Result; }
catch (AggregateException ae) { /* drill into ae.InnerException */ }
```

The `Task.Wait` MS Learn docs are explicit: *"`AggregateException`: The task was canceled. The `InnerExceptions` collection contains a `TaskCanceledException` object. -or- An exception was thrown during the execution of the task. The `InnerExceptions` collection contains information about the exception or exceptions."* This includes cancellation — a `Wait()` on a canceled task throws `AggregateException` wrapping `TaskCanceledException`, not the OCE you'd catch with `await`.

#### `GetAwaiter().GetResult()` — the lesser evil

If you genuinely cannot make the entry point async (a synchronous interface you don't control, a `Main` before C# 7.1, an event handler signature), `task.GetAwaiter().GetResult()` is the *least bad* option. It still blocks the thread (so starvation and deadlock risks remain), but it **unwraps exceptions the same way `await` does** — `InvalidOperationException` is `InvalidOperationException`, not `AggregateException` wrapping it. This is what `await` itself uses internally, so behavior is consistent.

```csharp
// Forced sync entry point — last resort
public string GetData() => GetDataAsync().GetAwaiter().GetResult();
```

Deeper coverage of state and exception unwrapping: [Task Lifecycle](11-task-lifecycle.md).

### async void — avoid it

```csharp
// WRONG
async void ProcessData() { ... }

// CORRECT - use async Task
async Task ProcessData() { ... }
```

Three real problems with `async void`:

1. **Unhandled exceptions crash the process** — they are raised on the captured `SynchronizationContext` (or `ThreadPool`), bypassing the caller's `try/catch`.
2. **Cannot be `await`ed** — there is no `Task` to observe completion, success, or failure.
3. **Breaks composition** — no way to `Task.WhenAll`, chain, cancel, or test reliably.

The only legitimate use is **UI event handlers** (the framework signature requires `void`).

### ConfigureAwait(false)

By default, `await` captures the current `SynchronizationContext` and resumes the continuation on it. `ConfigureAwait(false)` tells the runtime **not to capture** the context — the continuation runs on any thread pool thread.

```csharp
var data = await httpClient.GetStringAsync(url).ConfigureAwait(false);
```

- **Library code:** use `ConfigureAwait(false)` everywhere. You don't know who calls you, and forcing resumption on a caller's sync context causes perf loss and the deadlock above.
- **ASP.NET Core / console apps:** it's a **no-op** (no sync context to capture), but harmless.
- **App code in WinForms/WPF:** do NOT use it when you need to touch UI after the await.

### Fire-and-forget

An unawaited `Task` is **fire-and-forget** — nobody is watching for its completion or its exceptions.

```csharp
// BAD — if DoWorkAsync throws, no one catches it here
DoWorkAsync();
```

Three problems:

1. **Exceptions are easy to lose.** A faulted task's unobserved exception only surfaces via `TaskScheduler.UnobservedTaskException` when the runtime is about to trigger exception escalation policy — the timing is delayed and unpredictable, and the event is easy to miss in production.
2. **Shutdown can cut it mid-flight.** No one knows when it finished, so graceful shutdown may terminate the process before the work completes.
3. **`async void` is the worst variant.** Exceptions thrown from an `async void` method are re-raised on the `SynchronizationContext` that was active when the method started, bypassing the caller's `try`/`catch`. On a UI or classic ASP.NET context this can crash the app.

If fire-and-forget is genuinely what you want, wrap it and log:

```csharp
_ = Task.Run(async () =>
{
    try { await DoWorkAsync(); }
    catch (Exception ex) { _logger.LogError(ex, "Background work failed"); }
});
```

Better: enqueue into a `Channel<T>` or a hosted `BackgroundService` that owns lifecycle and error handling.

## `Task.FromResult` / `Task.CompletedTask` and "fake async"

`Task.FromResult<T>(value)` returns a `Task<T>` that is **already in `RanToCompletion`** — per the MS Learn docs, it "creates a `Task<TResult>` object whose `Task<TResult>.Result` property is `result` and whose `Status` property is `RanToCompletion`." `Task.CompletedTask` is the equivalent for non-generic `Task`. Both are useful — and both are abused.

### Legitimate uses

**1. Async interface with a synchronous fast path.** When *some* implementations of an interface need `await` (DB hit) but others can answer instantly (cache hit), wrap the instant value:

```csharp
public Task<User> GetByIdAsync(int id)
{
    if (_cache.TryGetValue(id, out var user))
        return Task.FromResult(user);   // hit: ready immediately, no allocation if cached singleton

    return LoadFromDbAsync(id);          // miss: real async
}
```

**2. Implementing a `Task`-returning interface from synchronous code.**

```csharp
public Task PublishAsync(Event evt)
{
    _localBus.Publish(evt);
    return Task.CompletedTask;    // method signature requires Task; nothing to actually await
}
```

**3. Mocks and test stubs** that need to satisfy an async signature without doing async work.

> .NET 6+ note: per the docs, "for some `TResult` types and some result values, this method may return a cached singleton object rather than allocating a new object." `Task.FromResult(true)`, `Task.FromResult(0)`, etc. may be allocation-free.

### The fake-async anti-pattern

The misuse is wrapping work that **never had any async** in `Task.FromResult` just because something downstream takes `Task<T>`:

```csharp
// ANTI-PATTERN — synchronous work behind an async signature
public Task<int> SumAsync(int[] nums) => Task.FromResult(nums.Sum());
```

The signature lies: it tells callers "this might suspend; await me." The caller pays state-machine + continuation costs and gains nothing because the work is instant and synchronous. When this becomes a convention ("everything is async for uniformity"), real async I/O becomes indistinguishable from facade.

The other classic fake-async is **`Task.Run(() => Sync())`** hidden inside library methods — it doesn't free a thread, it just shifts the work to a different one. Rule: expose synchronous work as synchronous; expose genuine async (real I/O / `TaskCompletionSource`) as async. Let callers decide when to offload.

### Hot-path alternative: `ValueTask<T>`

`Task.FromResult` still allocates a `Task<T>` (when the cached singleton path doesn't apply). For a hot synchronous fast path, return `ValueTask<T>` instead — a struct that wraps either the synchronous value (no allocation) or a real `Task<T>`. See [Async and Memory](../04-memory-and-performance/06-async-and-memory.md) for the rules.

## When synchronous beats async

The reflex "make everything async" is wrong as often as it's right. Async has costs: state machine allocation, continuation scheduling, uglier stack traces, `ExecutionContext` capture, and the viral "async all the way up" effect. The right question isn't "*can* this be async?" — it's "**what do I gain by freeing this thread?**" If the answer is nothing, async is just overhead.

Sync wins by default in these scenarios:

| Scenario | Why sync is fine |
|---|---|
| Pure CPU work (parsing, hashing, math) with no I/O | No wait to free up a thread for. Wrapping in `Task` is overhead. |
| Single-shot CLI tools, scripts, migrations | No "other users" to serve while you wait. The process has one job. |
| Naturally synchronous library code (parsers, validators, formatters) | Forcing `Async` suffixes onto things that don't do I/O breaks the contract. `JsonSerializer.Deserialize` is sync — and that's correct. |
| Hot paths where the state-machine cost > the work | Same reasoning as `ValueTask`, taken further. |
| Initialization that runs once at startup | No one is waiting; simplicity beats scalability. |

Async wins when **someone could be using this thread while you wait**: web servers (always another request), UI apps with I/O (blocking the UI thread freezes the app), batch processes with many concurrent I/O operations.

> Heuristic: **sync by default if no I/O, or if the process is single-shot. Async if there's I/O *and* concurrency (server, UI, parallel batch).**

## Realistic hybrid example

A single request often mixes both workloads. `await` each kind with the tool that fits it:

```csharp
public async Task<Report> GenerateReportAsync(int userId, CancellationToken ct)
{
    var data      = await _db.GetUserDataAsync(userId, ct);         // I/O — thread released
    var processed = await Task.Run(() => HeavyTransform(data), ct); // CPU — runs on a pool thread
    await _storage.SaveAsync(processed, ct);                        // I/O — thread released
    return new Report(processed);
}
```

Three `await`s, three distinct reasons to suspend. Only the middle one actually keeps a thread busy executing — the two I/O calls delegate the wait to the OS and return their thread to the pool in the meantime.

---

[← Previous: Parallel.ForEach and Invoke](02-parallel-foreach-invoke.md) | [Back to index](README.md) | [Next: Race Conditions →](04-race-conditions.md)
