# Memory Leak in C#

## What it is

A **memory leak** happens when the application allocates memory but fails to release it when it is no longer needed. Over time, this residual memory clogs the system, causing performance problems and, in the worst case, application crashes.

> In managed .NET, a "memory leak" really means **unintended rooted references**: the GC never frees objects that are still reachable from a root (static field, live thread stack, event handler, etc.). The fix is almost always to break the reference, not to free memory manually.

A **memory profiler** can be used to identify memory leaks.

## Common causes

### 1. Database connections not closed

Opening many database connections without closing them (inside a loop, for example).

```csharp
// WRONG - connection is never closed
for (int i = 0; i < 1000; i++)
{
    var conn = new SqlConnection(connectionString);
    conn.Open();
    // uses the connection but never closes it
}

// CORRECT - using ensures the connection is closed
for (int i = 0; i < 1000; i++)
{
    using var conn = new SqlConnection(connectionString);
    conn.Open();
    // connection automatically closed when leaving scope
}
```

### 2. Unregistered events

One of the most common causes of memory leaks in C# is forgetting to **unregister event handlers**. When you subscribe to an event, the object that owns the event handler keeps a reference to the subscriber, preventing garbage collection.

```csharp
public class Publisher
{
    public event EventHandler SomethingHappened;
}

public class Subscriber
{
    public void Subscribe(Publisher publisher)
    {
        publisher.SomethingHappened += HandleEvent;
    }

    public void HandleEvent(object sender, EventArgs e)
    {
        Console.WriteLine("Event handled.");
    }
}
```

**Solution:** always unregister from events when they are no longer needed.

```csharp
public void Unsubscribe(Publisher publisher)
{
    publisher.SomethingHappened -= HandleEvent;
}
```

### 3. Static references

Objects referenced by static variables can persist throughout the application's lifetime, and if not managed carefully, can lead to memory leaks.

```csharp
// PROBLEM: static list that only grows
public class MemoryLeak
{
    public static List<string> CachedData = new List<string>();
}
```

**Solution:** use weak references or ensure that static fields are cleared when no longer needed.

```csharp
// SOLUTION: method to clear the cache
public class MemoryLeak
{
    public static List<string> CachedData = new List<string>();

    public static void ClearCache()
    {
        CachedData.Clear();
    }
}
```

### 4. IDisposable objects not disposed

Objects that implement the `IDisposable` interface need proper cleanup. If the `Dispose` method is not called explicitly or we don't use the `using` block, it can lead to resource leaks and memory leaks.

**Wrong example:**
```csharp
using System;
using System.IO;

class Program
{
    static void Main()
    {
        FileStream file = new FileStream("example.txt", FileMode.Create);
        // Some work with the file...
        // We forget to call Dispose on 'file.'
    }
}
```

**Canonical dispose pattern (supports inheritance):**
```csharp
using System;

class Resource : IDisposable
{
    private bool _disposed;

    // Public, sealed entry point — never overridden by derived classes.
    public void Dispose()
    {
        Dispose(true);
        GC.SuppressFinalize(this);
    }

    // Derived classes override this to add their own cleanup.
    protected virtual void Dispose(bool disposing)
    {
        if (_disposed) return;

        if (disposing)
        {
            // Release managed resources here (other IDisposable fields).
        }

        // Release unmanaged resources here (raw handles, etc.).

        _disposed = true;
    }

    // Only needed if the class directly owns unmanaged resources.
    ~Resource() => Dispose(false);
}

class ChildResource : Resource
{
    private bool _disposed;

    protected override void Dispose(bool disposing)
    {
        if (_disposed) return;

        if (disposing)
        {
            // Release child-specific managed resources here.
        }

        _disposed = true;
        base.Dispose(disposing); // Always chain to the base.
    }
}

class Program
{
    static void Main()
    {
        ChildResource childResource = new ChildResource();
        try
        {
            // Some work with the childResource...
        }
        finally
        {
            // Ensure Dispose is called to release resources.
            childResource?.Dispose();
        }
    }
}
```

**Correct approach with using (recommended):**
```csharp
using System;
using System.IO;

class Program
{
    static void Main()
    {
        using (FileStream file = new FileStream("example.txt", FileMode.Create))
        {
            // Some work with the file...
        }
    }
}
```

### Use `IDisposable` correctly

Unmanaged resources (such as files, connections, streams, etc.) **are not automatically released by the GC**.

**Recommended practice:** implement and use `IDisposable`:

```csharp
using (var stream = new FileStream("data.txt", FileMode.Open))
{
    // safe usage
}
// This ensures resource release with Dispose().
```

### 5. Timers retaining `this`

`System.Threading.Timer` and `System.Timers.Timer` keep a strong reference to the callback. If the callback is an instance method, the timer keeps the **enclosing object** alive too — and everything the object references with it.

```csharp
public class DataRefresher : IDisposable
{
    private readonly Timer _timer;

    public DataRefresher()
    {
        _timer = new Timer(Refresh, null, 0, 1000);
        // The timer references Refresh, which references 'this'.
    }

    private void Refresh(object? state) { /* ... */ }

    public void Dispose() => _timer.Dispose();
}
```

If `Dispose` is forgotten, the entire object graph stays alive forever.

For new code, prefer `PeriodicTimer` (.NET 6+) — it is async-friendly, supports cancellation natively, and does not capture `ExecutionContext`:

```csharp
using var timer = new PeriodicTimer(TimeSpan.FromMinutes(1));
while (await timer.WaitForNextTickAsync(stoppingToken))
{
    await DoWorkAsync(stoppingToken);
}
```

### 6. AsyncLocal capture through ExecutionContext

`AsyncLocal<T>` itself does not leak — its values are tied to a **transient `ExecutionContext`** that becomes collectable when that context goes out of use. The leak surfaces when something **long-lived captures the `ExecutionContext`** at creation time: a `Timer`, a long-running `Task`, an event handler registered into a static event. That captured context keeps every `AsyncLocal` value present at capture time alive.

```csharp
private static readonly AsyncLocal<HeavyContext> _ctx = new();

public async Task ProcessRequestAsync()
{
    _ctx.Value = new HeavyContext { Data = new byte[10_000_000] };

    // Timer captures the current ExecutionContext, keeping _ctx.Value alive
    // for as long as the timer lives.
    var timer = new Timer(_ => DoStuff(), null, 0, 60_000);
    // If _timer is never disposed, the 10 MB stay rooted via the captured context.
}
```

Mitigations:

- Don't store heavy objects in `AsyncLocal` — prefer light identifiers / tokens
- Suppress `ExecutionContext` flow when creating long-lived objects:
  ```csharp
  using (ExecutionContext.SuppressFlow())
  {
      _timer = new Timer(callback, state, dueTime, period);
  }
  ```
- Clear the value explicitly at the end of the scope (`_ctx.Value = null`)

`PeriodicTimer` does **not** capture `ExecutionContext` and is the modern replacement for periodic callbacks in async code.

## Holding references without rooting them

When you need to refer to an object **without keeping it alive**, the BCL gives you two tools.

### `WeakReference<T>`

A weak reference points at an object **without preventing GC**. You ask, at access time, whether the object is still alive.

```csharp
var obj = new ExpensiveObject();
var weak = new WeakReference<ExpensiveObject>(obj);

obj = null;
GC.Collect();

if (weak.TryGetTarget(out ExpensiveObject? alive))
{
    // still alive — use it
}
else
{
    // collected — recreate or skip
}
```

Use cases: caches that must not block GC, breaking cycles in observer patterns, holding references in plugin systems where you don't own object lifetime.

For "real" caches (with TTL, size limits, eviction strategy), prefer `IMemoryCache` from `Microsoft.Extensions.Caching.Memory` over hand-rolled weak-reference structures.

### `ConditionalWeakTable<TKey, TValue>`

A specialized table where:

- **Keys are weak** (do not block their collection)
- **Values stay alive while the key is alive** — when the key is collected, the value entry disappears automatically
- **Both `TKey` and `TValue` must be reference types** (per the official MS Learn docs)
- **Equality is by reference identity only** — overrides of `Equals`/`GetHashCode` on the key are ignored

```csharp
private static readonly ConditionalWeakTable<HttpRequest, RequestMetrics> _metrics = new();

void OnRequestStart(HttpRequest req)
    => _metrics.Add(req, new RequestMetrics { StartTime = DateTime.UtcNow });

void OnRequestEnd(HttpRequest req)
{
    if (_metrics.TryGetValue(req, out var metrics))
        Log.Info($"Request took {DateTime.UtcNow - metrics.StartTime}");
    // No manual cleanup needed: when 'req' is collected, the entry is removed.
}
```

This is the right tool when you want to **attach data to objects you don't own** (third-party types, framework objects) without holding them alive yourself. WPF uses it for attached properties; the DLR uses it for expandos.

A naive `Dictionary<HttpRequest, RequestMetrics>` would keep every key alive forever — which is the leak this type was designed to avoid.

---

[← Previous: Memory Optimization](03-memory-optimization.md) | [Back to index](README.md) | [Next: Structs vs Classes →](05-structs-vs-classes.md)
