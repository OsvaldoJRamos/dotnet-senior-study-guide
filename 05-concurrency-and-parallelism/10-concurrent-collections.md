# Concurrent Collections

`List<T>`, `Dictionary<TKey, TValue>`, `Queue<T>`, and `Stack<T>` are **not thread-safe**. Concurrent mutation can corrupt the internal arrays or linked structures: lost items, duplicated entries, infinite loops on resize, `IndexOutOfRangeException` thrown from a method that "shouldn't" throw it. The fix is either external synchronization (correct but serializes everything) or one of the collections in `System.Collections.Concurrent`, which were designed from the start for concurrent access.

> For the catalog of which collection to pick when (concurrent and otherwise), see [Collections Overview](../02-collections-and-linq/04-collections-overview.md). This file focuses on **how the concurrent collections work** and the gotchas a senior is expected to know.

## Three strategies for thread safety

Every concurrent collection picks one (or mixes) of these:

| Strategy | How it works | Trade-off |
|---|---|---|
| **Coarse external lock** | Single lock around every operation | Correct, but serializes all access. The `ConcurrentDictionary` competitor when you wrap a `Dictionary` in `lock`. |
| **Lock striping (fine-grained locking)** | Multiple locks, each covering a partition of the data | Threads on different partitions don't contend. |
| **Lock-free** | `Interlocked.CompareExchange` and memory ordering instead of locks | Highest throughput under contention; hardest to get right. |

`ConcurrentDictionary` uses lock striping. `ConcurrentQueue` and `ConcurrentStack` are lock-free in their hot paths. `ConcurrentBag` uses thread-local state plus stealing. Different shapes for different access patterns.

## `ConcurrentDictionary<TKey, TValue>`

The official remarks set the contract:

> "Represents a thread-safe collection of key/value pairs that can be accessed by multiple threads concurrently."
>
> "For modifications and write operations to the dictionary, `ConcurrentDictionary<TKey, TValue>` uses fine-grained locking to ensure thread safety."
>
> "(Read operations on the dictionary are performed in a lock-free manner.)" — `ConcurrentDictionary<TKey, TValue>` MS Learn

Mental model:

- **Writes (`TryAdd`, `TryRemove`, indexer set) acquire one of N stripe locks** chosen by the key's hash, not a global lock. Two writes for keys in different stripes proceed in parallel.
- **Reads (`TryGetValue`, `ContainsKey`, indexer get, enumeration) take no lock.** They use volatile reads on the bucket, so they observe a consistent value but never block writers.

The `concurrencyLevel` constructor parameter sets how many stripes the table uses. The MS docs do not pin the default to a specific number — it varies by runtime version and is auto-tuned. The practical advice is to leave it alone unless you've benchmarked a hotspot.

### The factory-runs-multiple-times pitfall

`GetOrAdd`, `AddOrUpdate`, and `GetOrAdd<TArg>(...)` accept value-factory delegates. The official remarks are blunt about how they're invoked:

> "delegates for these methods are called outside the locks to avoid the problems that can arise from executing unknown code under a lock. Therefore, the code executed by these delegates is not subject to the atomicity of the operation."

In English: under contention, **two threads can both run the factory** for the same key, and only one of the produced values "wins" (gets stored). The other is silently discarded.

```csharp
// BAD — factory has side effects
_cache.GetOrAdd(key, k =>
{
    _logger.LogInformation("Loading {Key}", k);   // may log twice for the same key
    return _db.Load(k);                            // may hit the DB twice
});
```

The fix is to make the factory:

1. **Cheap and idempotent** (preferred), or
2. **Wrapped in a `Lazy<T>`** so the expensive part runs at most once:

```csharp
private readonly ConcurrentDictionary<string, Lazy<Config>> _cache = new();

public Config Get(string key) =>
    _cache.GetOrAdd(key, k => new Lazy<Config>(() => LoadFromDb(k))).Value;
```

Multiple threads may all create a `Lazy<Config>`, but only one wins the dictionary slot, and `Lazy<T>` itself ensures `LoadFromDb` runs at most once.

### Atomicity boundary

> "All these operations are atomic and are thread-safe with regards to all other operations on the `ConcurrentDictionary<TKey, TValue>` class." — MS Learn

The atomicity is **per operation**, not per logical task. If your invariant spans two keys ("decrement A and increment B"), `ConcurrentDictionary` won't help — you need a lock around the pair, or a redesign that fits a single key.

### When NOT to use it

- Read-heavy workload with rare writes — a `ImmutableDictionary<TKey, TValue>` swapped via `Interlocked.Exchange` is faster (every read sees a stable snapshot, no volatile reads).
- Read-only static lookup populated at startup — `FrozenDictionary<TKey, TValue>` (.NET 8+) is faster on lookup, slower to build.

## `ConcurrentQueue<T>`

> "Represents a thread-safe first in-first out (FIFO) collection." — MS Learn

The MS Learn API page does not detail the internal data structure — for the implementation, see `ConcurrentQueue.cs` in dotnet/runtime, which composes the queue from a series of internal segments rather than a single resizing array. Practical consequence: capacity grows without copying old contents on resize (unlike `Queue<T>`), and the common-case `Enqueue`/`TryDequeue` paths avoid taking locks.

Idiomatic drain pattern:

```csharp
while (_queue.TryDequeue(out var item))
    Process(item);
```

`Count` and enumeration take a snapshot — neither is a hot path operation. Don't loop on `Count` to dequeue; just call `TryDequeue` until it returns false.

## `ConcurrentStack<T>`

Lock-free LIFO. `Push` and `TryPop` use `Interlocked.CompareExchange` on the head pointer:

```text
push(item):
    do
        oldHead = _head
        newNode = Node(item, next: oldHead)
    while CAS(ref _head, newNode, oldHead) fails
```

Read, compute, swap. Retry on conflict. This is the canonical lock-free pattern (CAS retry loop).

`PushRange`/`TryPopRange` exist for batch operations and amortize the CAS cost when you have many items at once.

## `ConcurrentBag<T>`

> "`ConcurrentBag<T>` is a thread-safe bag implementation, optimized for scenarios where the same thread will be both producing and consuming data stored in the bag." — MS Learn

Each thread has its own local list:

- `Add(item)` writes to the **local** list — zero contention with other threads.
- `TryTake(out item)` reads from the **local** list first; if empty, it **steals** from another thread's list.

This shape fits *embarrassingly parallel* workloads where each thread accumulates results, possibly draining its own results, with the rare cross-thread steal. It does **not** fit classic producer/consumer patterns (one thread produces, a different thread consumes) — every consume becomes a steal, and stealing is more expensive than the dedicated path in `ConcurrentQueue`. Use `ConcurrentQueue` or `Channel<T>` for that.

## `BlockingCollection<T>`

A wrapper around any `IProducerConsumerCollection<T>` (defaults to `ConcurrentQueue<T>`) that adds:

- Optional **bounded capacity**.
- **Blocking semantics**: `Take()` blocks the thread when empty; `Add()` blocks when full.
- `CompleteAdding()` to signal the producer is done.
- `GetConsumingEnumerable()` for `foreach` consumption that exits on completion.

The catch: it's **synchronous**. `Take()` parks an entire OS thread waiting for an item. In modern async pipelines, prefer `Channel<T>` (covered in [Channels](09-channels.md)) — it offers the same shape async-first.

`BlockingCollection<T>` still earns its keep in legacy synchronous code or short-running console tools where the simplicity of "block until ready" matches the call site.

## Immutable collections (`System.Collections.Immutable`)

Mutations return a **new** collection; the original is unchanged. Internally, the implementations use **structural sharing** — most of the underlying tree is shared between versions, so a "mutation" only allocates the changed path.

```csharp
ImmutableList<string> tags = ImmutableList.Create("eu", "prod");
ImmutableList<string> more = tags.Add("beta");
// tags still has 2 items; more has 3; most internal nodes are shared
```

Why this matters for concurrency:

- **Reads are trivially thread-safe.** Two threads reading the same `ImmutableList<T>` see the exact same data and need no synchronization.
- **Snapshots are free.** Capturing "the state at this moment" is just holding the reference.
- **Writes are coordination via reference swap.** Use `Interlocked.CompareExchange` on a field of type `ImmutableDictionary<...>` to atomically install a new version.

The trade-off: every "mutation" allocates. For write-heavy workloads, `ConcurrentDictionary` wins. For read-heavy snapshots, immutables shine.

`FrozenDictionary<TKey, TValue>` and `FrozenSet<T>` (.NET 8+) are a different breed: built once, optimized purely for lookup speed, no mutations. Best for static reference data populated at startup and read forever.

## Quick decision rule

| Workload | Pick |
|---|---|
| Multi-threaded key/value cache, mixed reads and writes | `ConcurrentDictionary<TKey, TValue>` |
| Thread-safe FIFO between threads | `ConcurrentQueue<T>` |
| Async producer/consumer with backpressure | `Channel<T>` ([Channels](09-channels.md)) |
| Sync producer/consumer in legacy code | `BlockingCollection<T>` |
| Each thread accumulates its own results, occasional cross-thread drain | `ConcurrentBag<T>` |
| Read-heavy, snapshot semantics, occasional rebuild | `ImmutableDictionary<TKey, TValue>` swapped via `Interlocked.CompareExchange` |
| Static lookup table, build once, read forever | `FrozenDictionary<TKey, TValue>` (.NET 8+) |
| Thread-safe LIFO | `ConcurrentStack<T>` |

## Senior-interview gotchas

- **`ConcurrentDictionary` reads are lock-free; writes use lock striping.** Different stripes → no contention. Same stripe → serialized.
- **`GetOrAdd` and `AddOrUpdate` factories run outside the lock and may run multiple times** under contention. Wrap expensive factories in `Lazy<T>`.
- **Per-operation atomicity ≠ multi-operation atomicity.** "Decrement A, increment B" still needs a lock if both keys must move together.
- **`ConcurrentBag` is for "same thread produces and consumes."** Wrong tool for distinct producer/consumer threads — use `ConcurrentQueue` or `Channel<T>`.
- **`BlockingCollection.Take()` blocks an OS thread.** In async code, use `Channel<T>` instead.
- **Immutable collections + `Interlocked.CompareExchange`** is a clean pattern for "multiple readers, occasional rebuild" — every reader sees a consistent snapshot at zero synchronization cost.
- **Don't loop on `Count`** for any of these. The count is a snapshot; the FIFO/LIFO operations have their own "is it empty" semantics via `Try*` methods.

## Useful Links

- [`ConcurrentDictionary<TKey, TValue>` — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentdictionary-2) — fine-grained locking, lock-free reads, factory-outside-lock guidance
- [`ConcurrentQueue<T>` — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentqueue-1)
- [`ConcurrentBag<T>` — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.collections.concurrent.concurrentbag-1) — explicit "same thread produces and consumes" guidance
- [`BlockingCollection<T>` — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.collections.concurrent.blockingcollection-1)
- [`System.Collections.Immutable` — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.collections.immutable)

---

[← Previous: Channels](09-channels.md) | [Back to index](README.md) | [Next: Task Lifecycle →](11-task-lifecycle.md)
