# The Memory Model

When you write `_count++` in a method that runs on multiple threads, your code is making implicit assumptions about how memory works that are wrong. The CPU, the JIT, and the C# compiler are all allowed to **reorder** memory operations and keep values in registers — and they will, unless you tell them not to. The CLR memory model is the formal contract that says exactly which reorderings are legal and which primitives forbid them.

> This file focuses on the **conceptual model** — visibility, ordering, and the patterns built on top. For the BCL primitives that implement these guarantees (`lock`, `Monitor`, `Interlocked`, `Volatile`, `volatile`, `SemaphoreSlim`), see [Locks in Depth](../06-concurrency-and-parallelism/07-locks-in-depth.md).

## Two distinct problems

Concurrent code has to solve **mutual exclusion** (only one thread mutates the state at a time) and **memory ordering** (writes from one thread become visible to others in the order you intended). They sound similar but they're independent.

| Problem | What goes wrong | Tools |
|---|---|---|
| **Mutual exclusion** | Two threads modify the same field simultaneously, last-writer-wins corrupts state | `lock`, `Monitor`, `SemaphoreSlim`, `Mutex`, `Interlocked` for single-field ops |
| **Memory ordering** | Thread B sees stale values, or sees writes in the wrong order | `volatile`, `Volatile.Read`/`Volatile.Write`, full barriers from `lock`/`Interlocked` |

A `lock` solves both — that's why it's the right default. The other primitives let you pay only for what you need when locks are too heavy.

## What the runtime is allowed to do

The CLR memory model (ECMA-335 plus .NET refinements) guarantees that **single-threaded behavior is preserved**. Any reordering the compiler, JIT, or CPU can do that doesn't change what a single thread observes is fair game. Across threads, the rules are looser:

```csharp
// Source code:
_data = LoadData();
_isReady = true;

// What another thread can observe (without barriers):
//   _isReady == true && _data == null
```

There are three layers of reordering that can produce this result:

1. **Compiler reordering** — the C# compiler can move statements as long as the same thread sees consistent results.
2. **JIT reordering** — the JIT can move operations during code generation.
3. **CPU reordering** — modern CPUs (especially weakly-ordered ones like ARM) can reorder loads and stores at runtime via store buffers and out-of-order execution.

x86/x64 has a relatively strong memory model that forbids most reorderings, which is why bugs sometimes hide in development on Intel and surface in production on ARM. Don't rely on that — the CLR model is the contract, not the CPU.

## Visibility — separate from atomicity

A 32-bit aligned read or write is atomic on practically all hardware .NET supports. That guarantees the value you see is *some* full value the variable held, not a torn half-and-half. It does **not** guarantee the value is the most recent one another thread wrote.

```csharp
private bool _stopRequested;

// Thread A:
_stopRequested = true;

// Thread B:
while (!_stopRequested) { /* work */ }
// Without a barrier, Thread B can spin forever even after A wrote true.
// The write may sit in A's store buffer; the JIT may have hoisted the
// read of _stopRequested out of the loop.
```

This is **not a tearing bug**. The bool read is atomic. The problem is that the read is allowed to see an old value indefinitely. The fix is `volatile bool` or `Volatile.Read(ref _stopRequested)` — both insert the memory fences that force the read to actually go to memory.

## Acquire-release semantics

`Volatile.Read` and `Volatile.Write` (and the `volatile` keyword) implement **acquire-release**:

- **`Volatile.Read` (acquire fence)** — operations *after* this read cannot be reordered to *before* it.
- **`Volatile.Write` (release fence)** — operations *before* this write cannot be reordered to *after* it.

This is exactly enough to publish data safely to other threads:

```csharp
// Thread A (publisher):
_data = LoadData();              // (1)
Volatile.Write(ref _isReady, true);  // (2) release — (1) cannot move past (2)

// Thread B (consumer):
if (Volatile.Read(ref _isReady))     // (3) acquire — (4) cannot move before (3)
{
    Process(_data);              // (4)
}
```

The release on the writer pairs with the acquire on the reader: when B sees `_isReady == true`, the assignment to `_data` is guaranteed to have already happened from B's perspective.

A `lock` block is a stronger fence — it's a **full barrier** at both the acquire and release. Operations cannot move into or out of the locked region. That's the simplest correct model and the right default.

## Pattern: lock-free counter

A plain `_count++` is **three operations**: read, add, write. Even if each is atomic individually, they aren't atomic *together* — two threads can both read 41, both add, both write 42, losing one increment.

`Interlocked.Increment` issues a single CPU instruction (`LOCK XADD` on x86, `LDADD` on ARM 8.1) that does all three atomically without acquiring a lock object:

```csharp
private long _count;

public void OnRequest() => Interlocked.Increment(ref _count);
public long Total() => Interlocked.Read(ref _count);
```

`Interlocked.Read` is needed for `long` because on 32-bit runtimes a plain read of a 64-bit field is **not** atomic — it can tear. `Interlocked.Read` issues an atomic 8-byte read that doesn't tear regardless of bitness.

## Pattern: lock-free CAS retry loop

`Interlocked.CompareExchange` is the building block of every lock-free algorithm. It atomically *checks* that a field still holds the expected value and *only then* writes the new value, returning the original value either way.

```csharp
public int IncrementCapped(int max)
{
    int current;
    int next;
    do
    {
        current = _value;
        if (current >= max) return current;     // can't go higher
        next = current + 1;
    }
    while (Interlocked.CompareExchange(ref _value, next, current) != current);
    // CAS failed: another thread changed _value between read and write — retry.
    return next;
}
```

The pattern is: read, compute, attempt to commit with CAS, retry on conflict. This is how thread-safe in-memory counters, caches, and lock-free queues are built. It's also how `ConcurrentDictionary` updates entries internally.

## Pattern: double-checked locking

A textbook anti-pattern made correct only by understanding the memory model:

```csharp
private static MyService? _instance;
private static readonly object _lock = new();

public static MyService Instance
{
    get
    {
        // Fast path — no lock if already initialized
        var local = Volatile.Read(ref _instance);
        if (local is not null) return local;

        lock (_lock)
        {
            // Re-check under the lock — another thread may have just initialized
            if (_instance is null)
            {
                _instance = new MyService();
            }
            return _instance;
        }
    }
}
```

The first read needs an acquire fence (`Volatile.Read`); without it, the JIT could hoist the read above the conditional and skip the check. The write inside the lock is implicitly a release because `lock` exits with a full barrier.

In practice, you should almost always **use `Lazy<T>` instead** — it implements this pattern correctly with `LazyThreadSafetyMode.ExecutionAndPublication` (the default), and the failure mode of getting it wrong by hand is silent.

```csharp
private static readonly Lazy<MyService> _instance = new(() => new MyService());
public static MyService Instance => _instance.Value;
```

## Pattern: stop-flag without a lock

The simplest case where the memory model bites you. A worker loop polling a flag:

```csharp
public class Worker
{
    private volatile bool _stop;     // ← keyword version

    public void Stop() => _stop = true;

    public void Run()
    {
        while (!_stop) { DoWork(); }   // each read is a volatile read
    }
}
```

The `volatile` keyword makes every read a volatile read and every write a volatile write — the field is forced to memory each time, never cached in a register. The cost is small: barriers are cheap. The cost of getting this wrong is a worker that ignores `Stop()` until the JIT happens to invalidate its register, which is non-deterministic.

`volatile` is restricted to types of size at most a native pointer — no `long` on 32-bit, no structs, no array elements. For those, use `Volatile.Read`/`Volatile.Write` at the call site:

```csharp
Volatile.Write(ref _stopArr[i], 1);
if (Volatile.Read(ref _stopArr[i]) == 1) { ... }
```

## When you need a full memory barrier

Rarely, but sometimes. `Thread.MemoryBarrier()` (or `Interlocked.MemoryBarrier()`) is a **full barrier** — no operation can move across it in either direction. It's heavier than a single acquire or release.

You need this when you need ordering in *both* directions, and `Volatile.Read`/`Volatile.Write` only give you one. Almost every scenario that needs this is better written with a `lock` instead. The barrier is the right tool when you're implementing low-level synchronization primitives yourself, which is rare in application code.

## Decision table

| Scenario | Pick |
|---|---|
| Anything compound (read-modify-write of a single field) involving multiple threads | `lock`, or `Interlocked.CompareExchange` for the simple cases |
| Single-field atomic counter / flag swap | `Interlocked.Increment` / `CompareExchange` |
| Stop flag, single-field read by many readers | `volatile` field or `Volatile.Read`/`Volatile.Write` |
| Lazy initialization | `Lazy<T>` |
| 64-bit field on potentially 32-bit runtime | `Interlocked.Read` / `Interlocked.Exchange` |
| Producer publishes data + flag for consumer | `Volatile.Write` after data assignment, `Volatile.Read` before reading data |
| Anything else cross-thread | Default to `lock`. Optimize only with profile evidence. |

## Senior-interview gotchas

- **Atomic ≠ thread-safe.** Atomic reads and writes prevent tearing; they don't prevent races on read-modify-write sequences.
- **`volatile` is field-level; `Volatile.Read`/`Volatile.Write` is operation-level.** New code should prefer the operation-level form — it's explicit at the call site and works on any field type.
- **`Volatile` guarantees ordering, not mutual exclusion.** Two threads doing `Volatile.Read` + modify + `Volatile.Write` still race.
- **`lock` is a full barrier.** Code inside `lock` sees a consistent view of memory written by the previous holder. Outside locks, you need explicit barriers.
- **x86 hides bugs.** Code that's "fine" in dev on x86 can break on ARM in production because ARM has a weaker memory model. Reason about the CLR model, not what the CPU happens to allow.

---

[← Previous: Pipelines](09-pipelines.md) | [Back to index](README.md) | [Next: Modern String APIs →](11-modern-string-apis.md)
