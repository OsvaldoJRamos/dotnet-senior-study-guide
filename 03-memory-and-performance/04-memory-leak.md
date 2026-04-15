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

---

[← Previous: Memory Optimization](03-memory-optimization.md) | [Back to index](README.md) | [Next: Structs vs Classes →](05-structs-vs-classes.md)
