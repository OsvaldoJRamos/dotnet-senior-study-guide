# Locks in Depth

The `lock` statement is the first primitive most C# developers reach for, but .NET ships a whole toolbox of synchronization primitives. Picking the wrong one is a common senior-interview gotcha. This file covers what `lock` actually compiles to, the new `System.Threading.Lock` in .NET 9+, and when to reach for `Monitor`, `ReaderWriterLockSlim`, `Mutex`, `SpinLock`, `Interlocked`, or `Volatile` instead.

## What `lock` actually compiles to

The C# compiler emits different code depending on the type of the lock target.

### Case A: target is `object` (or any reference type that isn't `System.Threading.Lock`)

```csharp
lock (_lockObj) { /* body */ }
```

Lowers to `Monitor.Enter`/`Monitor.Exit` inside a `try/finally`:

```csharp
object __lockObj = _lockObj;
bool __lockWasTaken = false;
try
{
    System.Threading.Monitor.Enter(__lockObj, ref __lockWasTaken);
    // body
}
finally
{
    if (__lockWasTaken) System.Threading.Monitor.Exit(__lockObj);
}
```

The `ref bool __lockWasTaken` pattern guarantees the lock is only released if it was actually taken, even if `Monitor.Enter` was interrupted by a `ThreadAbortException` between acquiring the lock and the `try` block.

### Case B: target is `System.Threading.Lock` (.NET 9 / C# 13+)

```csharp
private readonly System.Threading.Lock _lockObj = new();

lock (_lockObj) { /* body */ }
```

Lowers to the new `EnterScope` pattern:

```csharp
using (_lockObj.EnterScope())
{
    // body
}
```

`EnterScope()` returns a `ref struct` (`Lock.Scope`) whose `Dispose()` releases the lock. Same `try/finally` semantics, but avoids going through the object header and Monitor infrastructure — measurably faster under contention in the Microsoft benchmarks.

> The compiler only emits the new pattern when the **static type** of the expression is exactly `System.Threading.Lock`. If you assign a `Lock` to an `object` or a generic `T` and lock on it, the compiler falls back to `Monitor.Enter`. C# 13 also raises a warning if you cast a known `Lock` to another type and lock it.

## `System.Threading.Lock` (.NET 9+)

A dedicated lock type that replaces the old idiom of `private readonly object _lockObj = new();`.

```csharp
public sealed class OrderProcessor
{
    private readonly System.Threading.Lock _gate = new();
    private int _processed;

    public void Process(Order order)
    {
        lock (_gate)
        {
            _processed++;
            // ... do work
        }
    }
}
```

Key members:

| Member | Purpose |
|---|---|
| `Enter()` / `Exit()` | Blocking acquire / release. Exit must be called on the same thread that entered. |
| `EnterScope()` | Returns a `ref struct` for `using` — the idiomatic form. |
| `TryEnter()` / `TryEnter(int)` / `TryEnter(TimeSpan)` | Non-blocking or bounded-wait acquire. |
| `IsHeldByCurrentThread` | For diagnostics and assertions. |

Why prefer it over locking on `object`:

1. **Intent is explicit** — the type literally says "this is a lock".
2. **Performance** — avoids object-header-based thin-lock machinery.
3. **Tooling** — analyzers can reason about it; the compiler warns on misuse.
4. **No collisions** — nobody else can `Monitor.Enter` on it through an `object` reference.

## Why not lock on `this`, `typeof(X)`, or string literals

These are **shared references** that anyone else can also lock on — causing lock contention or deadlock between unrelated code.

| Bad target | Why it's dangerous |
|---|---|
| `this` | Callers can write `lock (myInstance) { ... }` from outside and serialize against your internal lock. |
| `typeof(MyClass)` | The `Type` object is process-wide and can be obtained by any code via `typeof` or reflection. Locking on it creates hidden cross-assembly contention. |
| String literals | Strings are **interned**. `lock ("key")` in two unrelated libraries locks on the same reference. |

Concrete risk: a library locks on `this`, and a caller wraps the library in another `lock (libraryInstance)` for their own reasons — any method the library calls back into the caller (event, virtual, delegate) can deadlock.

**Rule:** always lock on a dedicated, `private readonly` field. With `System.Threading.Lock` this is trivial; with `object` it's still the standard idiom.

## `Monitor` — what `lock` uses under the hood (pre-.NET 9 / non-`Lock` targets)

`Monitor` is a static class around the CLR's per-object lock. The key facts:

- **Thread-affine**: the same thread that called `Monitor.Enter` must call `Monitor.Exit`. A different thread throws `SynchronizationLockException`.
- **Reentrant**: the holding thread can re-enter the same lock; you must `Exit` the same number of times.
- `Monitor.Pulse` / `Monitor.Wait` / `Monitor.PulseAll` implement a condition-variable pattern — rarely used directly in modern code; prefer `SemaphoreSlim`, `ManualResetEventSlim`, or channels.
- `Monitor.TryEnter(obj, timeout, ref bool)` gives you a bounded wait.
- `Monitor.LockContentionCount` (static counter) is useful for diagnosing hot locks.

```csharp
bool taken = false;
try
{
    Monitor.TryEnter(_lockObj, TimeSpan.FromSeconds(1), ref taken);
    if (!taken) throw new TimeoutException();
    // critical section
}
finally
{
    if (taken) Monitor.Exit(_lockObj);
}
```

> Thread affinity is also the reason `await` inside `lock` is forbidden — see the next section.

## `await` inside `lock` is a compile error (CS1996)

```csharp
lock (_gate)
{
    await database.SaveAsync(); // CS1996
}
```

The compiler refuses because `Monitor` / `Lock` are **thread-affine**: the code after `await` may resume on a **different thread**, which cannot release the lock. Even if the runtime allowed it, you'd leak the lock forever and eventually deadlock other threads.

The async-friendly alternative is `SemaphoreSlim`, which is not thread-affine — any thread may call `Release`. See [Deadlocks](05-deadlocks.md) and [SemaphoreSlim](06-semaphore-slim.md).

## `ReaderWriterLockSlim` — many readers, one writer

Use when **reads vastly outnumber writes** and read work is non-trivial. Multiple threads can hold the read lock simultaneously; only one thread at a time can hold the write lock, and it blocks all readers.

```csharp
private readonly ReaderWriterLockSlim _rw = new(LockRecursionPolicy.NoRecursion);
private readonly Dictionary<string, Config> _cache = new();

public Config Get(string key)
{
    _rw.EnterReadLock();
    try { return _cache[key]; }
    finally { _rw.ExitReadLock(); }
}

public void Set(string key, Config value)
{
    _rw.EnterWriteLock();
    try { _cache[key] = value; }
    finally { _rw.ExitWriteLock(); }
}
```

Modes:

| Mode | Semantics |
|---|---|
| `EnterReadLock` | Multiple concurrent readers. Blocks while a writer holds the lock. |
| `EnterWriteLock` | Exclusive. Blocks readers and other writers. |
| `EnterUpgradeableReadLock` | Single upgradeable reader + multiple plain readers. Can be upgraded to a write lock without releasing — avoids the "read → release → write" race. Only **one** upgradeable reader at a time. |

`LockRecursionPolicy.NoRecursion` (the default) is the right choice unless you genuinely need recursion — recursive locks hide bugs.

> `ReaderWriterLockSlim` is heavier than `lock` for short critical sections. A single `lock` is often faster for quick reads. Benchmark before reaching for it. Also: for simple thread-safe lookups, `ConcurrentDictionary<TKey,TValue>` is usually a better answer than rolling your own reader-writer cache.

## `Mutex` — cross-process synchronization

`Mutex` is a **kernel object** (a `WaitHandle`). It's **much slower** than `lock`/`Monitor` (lock acquisition requires a kernel transition) but has one capability `lock` does not: it can be **named** and shared across processes.

Canonical use: **single-instance application** detection.

```csharp
// Name with "Global\" prefix for cross-session visibility on Windows
using var mutex = new Mutex(initiallyOwned: false, name: @"Global\MyApp.SingleInstance",
                             out bool createdNew);

if (!createdNew)
{
    Console.WriteLine("Another instance is already running.");
    return;
}

// The process holds the name; releasing the mutex on exit is automatic.
```

Key properties:

- Acquire via `WaitOne()` (from the inherited `WaitHandle`); release via `ReleaseMutex()`.
- Enforces thread identity — only the acquiring thread may release.
- If the owning thread exits without releasing, the mutex is **abandoned**; the next waiter throws `AbandonedMutexException`. Treat this as a sign of a crash — the protected data may be inconsistent.
- On Unix-like systems, named mutexes are backed by file-system primitives with user-level scoping gotchas — read the docs.

> For same-process synchronization always prefer `lock` or `System.Threading.Lock`. Reach for `Mutex` only when you genuinely need cross-process coordination.

## `SpinLock` — pure user-mode spin

A struct (value type!) that spins in a tight loop until the lock becomes available, never making a kernel call. Only beneficial for **extremely short** critical sections where the expected wait is shorter than the cost of a context switch.

```csharp
private SpinLock _spin = new(enableThreadOwnerTracking: false);

public void Increment(ref int counter)
{
    bool taken = false;
    try
    {
        _spin.Enter(ref taken);
        counter++;
    }
    finally
    {
        if (taken) _spin.Exit();
    }
}
```

Rules you must follow while holding a `SpinLock` (from the official docs):

- Don't block, don't call anything that could block.
- Don't hold more than one spin lock.
- Don't make virtual / interface / delegate calls.
- Don't allocate memory.

Pitfalls:

- `SpinLock` is a **struct**. Copying it creates an independent lock. **Never** store it in a `readonly` field — a `readonly` struct field is defensively copied by the compiler on each access, so `Enter` operates on a copy and does nothing useful. Always pass by `ref`.
- `enableThreadOwnerTracking: false` makes it faster but means `Exit` can be called from any thread — easy to corrupt.
- The docs explicitly say: only use `SpinLock` after benchmarks prove `lock` is a bottleneck.

## `SemaphoreSlim` — N concurrent + async-friendly

See [SemaphoreSlim](06-semaphore-slim.md). Short version: it allows up to N concurrent holders, is not thread-affine, and supports `WaitAsync` — making it the right choice whenever an `await` is involved.

## `Interlocked` — atomic single-variable ops

Not a lock. `Interlocked` provides CPU-level atomic operations on a single 32-bit, 64-bit, or native-sized field. Lock-free. No blocking, no thread affinity.

```csharp
private long _processed;

public void OnItem() => Interlocked.Increment(ref _processed);
public long Total() => Interlocked.Read(ref _processed);
```

Common operations:

| Method | Purpose |
|---|---|
| `Increment` / `Decrement` | Atomic `++` / `--`. |
| `Add` | Atomic `+=`. |
| `Exchange(ref target, newValue)` | Atomic swap, returns old value. |
| `CompareExchange(ref target, newValue, expected)` | CAS — the foundation of every lock-free algorithm. |
| `Read(ref long)` | Atomic read of a 64-bit field (important on 32-bit CPUs where a plain read isn't atomic). |
| `MemoryBarrier()` | Full memory fence. |

**When it fits:** a counter, a flag, a pointer swap. **When it doesn't:** anything that requires updating two fields consistently — you need a real lock or a transactional structure.

`CompareExchange` is how you build lock-free stacks, once-only initializers, etc.:

```csharp
// One-time lazy initialization without a lock
private Config? _config;

public Config GetConfig()
{
    var current = Volatile.Read(ref _config);
    if (current is not null) return current;

    var fresh = LoadConfig();
    // Install only if nobody else beat us to it
    return Interlocked.CompareExchange(ref _config, fresh, null) ?? fresh;
}
```

## `Volatile` and the `volatile` keyword — memory barriers

Modern CPUs and the JIT can **reorder** reads and writes across threads for performance. A plain read of a shared field can see a stale value indefinitely, and writes can appear out of order to other threads.

```csharp
// WRONG — stopRequested write may never become visible to the spinning thread
private bool _stopRequested;

void Worker()
{
    while (!_stopRequested) { /* ... */ }
}

void Stop() => _stopRequested = true;
```

Two fixes:

### `volatile` keyword (field-level)

```csharp
private volatile bool _stopRequested;
```

Every read is a **volatile read** (prevents later ops from moving before it), every write is a **volatile write** (prevents earlier ops from moving after it). Limited to types of size at most a native word (no `long` on 32-bit, no structs, no array *elements*).

### `System.Threading.Volatile` (operation-level)

```csharp
private bool _stopRequested;

void Worker()
{
    while (!Volatile.Read(ref _stopRequested)) { /* ... */ }
}

void Stop() => Volatile.Write(ref _stopRequested, true);
```

Works on any field, including array elements and `long`/`double` on 32-bit systems (where `Volatile.Read`/`Write` also guarantee atomicity, unlike plain accesses). Prefer this over the keyword in new code — it's explicit at the call site.

> `Volatile` guarantees **ordering**, not **mutual exclusion**. Two threads doing `Volatile.Read` + modify + `Volatile.Write` still race. For anything beyond a single read or write, use `Interlocked` or a lock.

## The CLR memory model in one paragraph

The CLR memory model (ECMA-335 + .NET clarifications) allows the compiler, JIT, and CPU to **reorder** memory operations as long as **single-threaded** behavior is preserved. On multi-core hardware, thread A's writes can appear out of order to thread B. Every full lock acquire/release inserts a memory barrier, so code inside `lock` sees a consistent view of memory written by the previous holder. Outside of locks, you need `volatile`, `Volatile.Read`/`Write`, `Interlocked`, or an explicit `Interlocked.MemoryBarrier()` to control ordering.

## Rule-of-thumb: which primitive to pick

| Scenario | Use |
|---|---|
| Classic "one thread at a time" critical section | `lock` on a `System.Threading.Lock` (.NET 9+) or `object` |
| Same as above but **inside `async`** code | `SemaphoreSlim.WaitAsync(1,1)` — never `lock` |
| Atomic counter, flag, or pointer swap | `Interlocked.Increment` / `CompareExchange` |
| Thread-safe dictionary, queue, bag | `ConcurrentDictionary`, `ConcurrentQueue`, `ConcurrentBag`, `Channel<T>` |
| Reads vastly outnumber writes on a custom structure | `ReaderWriterLockSlim` (benchmark against a plain `lock` first) |
| Limit N concurrent callers (e.g., max 5 HTTP calls) | `SemaphoreSlim(5, 5)` |
| Cross-process single-instance app | named `Mutex` |
| Extremely short, high-frequency critical sections | `SpinLock` (only after profiling proves it's needed) |
| Cross-thread visibility of a single field (stop flag, etc.) | `volatile` field or `Volatile.Read`/`Volatile.Write` |
| Wake one / many waiting threads | `ManualResetEventSlim`, `AutoResetEvent`, `Channel<T>` |

## Common senior-interview gotchas

- **`lock` + `await` = CS1996.** Always. This is a compiler error, not a runtime bug. See [Deadlocks](05-deadlocks.md).
- **`Monitor` / `Lock` are thread-affine.** A different thread can't `Exit`. Design async code around primitives that aren't (`SemaphoreSlim`).
- **`SpinLock` in a `readonly` field is a silent bug** — the compiler defensively copies the struct on every access, so `Enter`/`Exit` don't operate on the same instance.
- **`Interlocked` operations are atomic, but composing two of them is not** — e.g., `if (Interlocked.Read(ref x) > 0) Interlocked.Decrement(ref x);` still races.
- **Locking on `typeof(X)` or interned strings** can deadlock unrelated code. Always use a dedicated private field.
- **`volatile` is not a substitute for a lock.** It prevents stale reads and reordering for a single field, but not compound updates.
- **`ReaderWriterLockSlim` is slower than `lock` for tiny critical sections.** Measure before you switch.
- **Database locks are a completely separate beast** — process-boundary, escalation, multiple granularities. See [Database Locks](../09-data-access/07-database-locks.md).

---

[← Previous: SemaphoreSlim](06-semaphore-slim.md) | [Back to index](README.md)
