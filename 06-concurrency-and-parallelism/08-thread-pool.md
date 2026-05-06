# The Thread Pool

`Task.Run`, `await`, `Parallel.ForEach`, `Timer` callbacks, async I/O completions — almost every concurrent primitive in .NET runs on the **thread pool**. Understanding what's underneath turns abstract advice ("don't block on async") into mechanical reasoning about *why* it breaks.

## Why a pool at all

Creating an OS thread is expensive: a reserved stack (the .NET default is 1 MB per the `Thread` constructor docs), a scheduler entry, a syscall. For tasks that finish in microseconds, the creation cost dwarfs the work. Running thousands of OS threads in parallel makes it worse — context switching destroys throughput.

The .NET thread pool solves both: a managed pool of pre-created, reusable threads that wait for work. From the official remarks:

> "Many applications create threads that spend a great deal of time in the sleeping state, waiting for an event to occur. ... The thread pool enables you to use threads more efficiently by providing your application with a pool of worker threads that are managed by the system." — `ThreadPool` docs

## Two pools: worker threads and I/O completion threads

`ThreadPool.GetMinThreads` and `GetMaxThreads` both return **two** numbers because the pool tracks two categories:

| Category | Used for |
|---|---|
| **Worker threads** | CPU work: `Task.Run`, `ThreadPool.QueueUserWorkItem`, `Timer` callbacks, registered wait callbacks |
| **I/O completion threads** | Completions of asynchronous I/O bound to the pool (on Windows: I/O Completion Ports / IOCP) |

The pool maintains separate min/max for each. Saturating one does not starve the other — a CPU-bound app stuck in `Task.Run` won't necessarily block I/O completion threads, and vice versa.

> The `BindHandle(SafeHandle)` API binds an OS handle to the thread pool so its async I/O completions are dispatched on completion-port threads. High-level APIs (`HttpClient`, `FileStream`, sockets, `SqlConnection`) already do this.

## How the pool sizes itself

From the docs:

> "Beginning with the .NET Framework 4, the thread pool creates and destroys worker threads in order to optimize throughput, which is defined as the number of tasks that complete per unit of time."

The runtime ships an algorithm called **hill climbing** for that adjustment. The actual implementation lives at `System.Private.CoreLib/src/System/Threading/PortableThreadPool.HillClimbing.cs` in the runtime; its file header reads:

> "Hill climbing algorithm used for determining the number of threads needed for the thread pool."

Mental model: the pool measures completed-work-per-second, nudges thread count up or down, and keeps the change that improved throughput. It runs continuously, so the optimal count adapts to changing workload mix.

### The `MinThreads` floor

> "By default, the minimum number of threads is set to the processor count." — `SetMinThreads` docs

Below `MinThreads`, the pool **creates threads on demand without delay** — request hits the queue, a thread is materialized immediately. Above the floor, hill climbing decides whether to add another, and additions are deliberately **rate-limited** so the pool doesn't stampede when a burst arrives. Concretely: a sudden flood of work briefly queues up while the pool decides to grow.

### When to call `SetMinThreads`

Almost never in modern code. The docs are explicit:

> "In most cases, the thread pool will perform better with its own algorithm for allocating threads. Reducing the minimum to less than the number of processors can also hurt performance."

The legitimate use case the docs call out:

> "to temporarily work around issues where some queued work items or tasks block thread pool threads. Those blockages sometimes lead to a situation where all worker or I/O completion threads are blocked (starvation)."

In other words, raising `MinThreads` is a band-aid for thread pool starvation while you fix the actual cause.

## Work stealing

A flat global queue would serialize the pool — every dequeue contends on one lock. Instead, the runtime gives each pool thread its own **local queue** for work it produced (e.g., a `Task.Run` called from inside another pool task), and uses the global queue for work coming from outside. An idle thread that finishes its own queue **steals** from another thread's local queue.

The official `SetMinThreads` remarks confirm the pattern when warning about over-provisioning:

> "Worker threads may take more CPU time in dequeuing work items due to having to scan more threads to **steal work** from."

Two consequences worth knowing:

1. **`Task.Run` on a pool thread is cheap.** The new task lands on the local queue, no global contention.
2. **Cache locality.** Work produced and consumed by the same thread tends to operate on hot data. The pool exploits that with LIFO ordering on the local queue (most recent work first).

## Thread pool starvation — the classic bug

Pool threads are a finite resource. Block enough of them and the pool runs dry; throughput collapses; the runtime tries to inject more, but rate-limited injection is slow.

The textbook trigger is **synchronous waits on async work**:

```csharp
// BAD — burns one pool thread to sit and block while the I/O is in flight.
var data = httpClient.GetStringAsync(url).Result;
```

What goes wrong:

1. The continuation after the network read needs a pool thread to resume.
2. The current thread is sitting on `.Result`, parked, waiting.
3. Multiply by N concurrent requests → N pool threads stuck → pool empty.
4. Hill climbing tries to add threads, but injection is throttled. Latency spikes for every request.

In ASP.NET Core, this manifests as request queues backing up, P99 latency exploding, then sometimes recovering 30+ seconds later as the pool slowly grows. You can see it directly in metrics:

```csharp
// EventCounter: "threadpool-queue-length" in System.Runtime
// or programmatically:
ThreadPool.GetMinThreads(out int minWorker, out _);
ThreadPool.GetMaxThreads(out int maxWorker, out _);
ThreadPool.GetAvailableThreads(out int availWorker, out _);
// busy = max - avail; if busy ≫ min and queue is growing, you're starving
```

**The fix is never `SetMinThreads`.** The fix is to be `async` end-to-end so the pool threads are released during I/O. `SetMinThreads` only buys time while you remove the blocking calls.

## Async I/O on the pool — what `await` actually does

Combining the worker pool with IOCP gives async I/O its scaling property:

1. Code calls `await stream.ReadAsync(...)`.
2. The OS / driver receives the read request; the read is in flight.
3. The state machine registers a continuation. **The worker thread is released back to the pool.**
4. When the device signals completion, an **I/O completion thread** picks up the completion.
5. The continuation is queued (typically to the worker pool) and runs on whichever pool thread picks it up.

This is why a .NET server can hold 10,000 sockets open with ~32 worker threads — the threads aren't sitting on the sockets, they're picking up continuations as completions arrive.

## Reusing pool threads — gotchas

> "When the thread pool reuses a thread, it does not clear the data in thread local storage or in fields that are marked with the `ThreadStaticAttribute` attribute." — `ThreadPool` docs

Two practical consequences:

1. **`[ThreadStatic]` and `ThreadLocal<T>` are dangerous on pool threads.** A previous request's value may still be there. For request-scoped state in async code, use `AsyncLocal<T>` instead — it flows through the logical execution context, not the physical thread.
2. **Pool threads are background threads** (`IsBackground = true`). They do not keep the process alive — if main returns and only pool threads are running, the process exits.

## Configuring vs replacing the pool

On Windows .NET 6+ you can opt into the **Windows OS thread pool** instead of the .NET portable pool:

```xml
<ItemGroup>
  <RuntimeHostConfigurationOption Include="System.Threading.ThreadPool.UseWindowsThreadPool" Value="true" />
</ItemGroup>
```

When enabled, `SetMinThreads` is **not supported** (the docs explicitly call this out in the `SetMinThreads` remarks). The Windows pool has its own scheduling and isn't tunable via the .NET APIs. Useful for some interop-heavy scenarios; rarely needed.

## Senior-interview gotchas

- **MinThreads default = logical processor count.** Beyond that, thread injection is rate-limited.
- **Hill climbing is throughput-based.** It nudges thread count and keeps the change if throughput improved. Implementation: `PortableThreadPool.HillClimbing.cs`.
- **Worker threads and I/O completion threads are separate pools.** Counting only one when reasoning about starvation gives the wrong picture.
- **`.Result` / `.Wait()` is the canonical starvation cause.** The fix is async-all-the-way, not `SetMinThreads`.
- **Local queue + global queue + work stealing.** `Task.Run` from inside the pool stays local; outside requests hit the global queue; idle threads steal.
- **`[ThreadStatic]` is unsafe on pool threads.** Pool reuse doesn't reset thread-local storage. Use `AsyncLocal<T>` for flowing state.
- **Pool threads are background threads.** They don't keep the process alive on their own.

## Useful Links

- [`ThreadPool` Class — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.threading.threadpool)
- [`ThreadPool.SetMinThreads` — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.threading.threadpool.setminthreads) — explicit guidance on when (rarely) to raise the floor and the performance trade-offs
- [Threading config settings — MS Learn](https://learn.microsoft.com/en-us/dotnet/core/runtime-config/threading) — Windows thread pool, GC threading
- [`PortableThreadPool.HillClimbing.cs`](https://github.com/dotnet/runtime/blob/main/src/libraries/System.Private.CoreLib/src/System/Threading/PortableThreadPool.HillClimbing.cs) — the hill-climbing implementation in dotnet/runtime

---

[← Previous: Locks in Depth](07-locks-in-depth.md) | [Back to index](README.md) | [Next: Channels →](09-channels.md)
