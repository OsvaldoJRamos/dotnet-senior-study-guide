# Deadlocks

## What is it

**Deadlock** is a situation in concurrent computing where two or more threads or processes are blocked, waiting for each other to release resources that each one needs to proceed, resulting in a circular wait. It can be difficult to detect and can lead to severe performance degradation or even system crashes.

## How to prevent

### 1. Avoid circular dependencies

Circular dependencies occur when two or more threads are waiting for resources held by each other. This can be avoided by designing the system so that each process acquires resources in a **specific order** and releases them in the **reverse order**.

### 2. Use Lock Hierarchy

Use a lock hierarchy to prevent circular dependencies. A lock hierarchy is a set of locks organized in a specific order, and each thread acquires locks in the same order.

```csharp
private object lock1 = new object();
private object lock2 = new object();

public void Method1()
{
    lock (lock1)
    {
        lock (lock2)
        {
            // Access the resources
        }
    }
}

public void Method2()
{
    lock (lock1)       // same order as Method1!
    {
        lock (lock2)
        {
            // Access the resources
        }
    }
}
```

`Method1` and `Method2` acquire locks `lock1` and `lock2` **in the same order**, ensuring there is no circular wait.

### 3. Do not mix `lock` with `await`

```csharp
// DOES NOT COMPILE - CS1996
lock (locker)
{
    await database.SaveAsync();
}
```

Two facts to remember:

1. **`await` inside a `lock` block is a C# compile error (CS1996).** The compiler forbids it outright.
2. Even if it compiled, `Monitor` (what `lock` uses under the hood) is **thread-affine**: only the thread that acquired the lock may release it. After an `await`, the continuation can resume on a different thread, which would then fail to release the monitor.

**Solution:** use `SemaphoreSlim.WaitAsync()` for async mutual exclusion.

### 4. TPL does NOT prevent deadlocks

The Task Parallel Library helps you avoid manual thread management (`Thread.Start`, manual pools), but it does **not** eliminate deadlocks or race conditions. Any shared state accessed from `Task.Run` still needs synchronization — that responsibility is yours. Do not assume using `Task` makes concurrent code safe.

### 5. Use timeouts

```csharp
bool lockTaken = false;
try
{
    Monitor.TryEnter(lockObj, TimeSpan.FromSeconds(5), ref lockTaken);
    if (lockTaken)
    {
        // resource acquired
    }
    else
    {
        // timeout - log and fallback
    }
}
finally
{
    if (lockTaken) Monitor.Exit(lockObj);
}
```

---

[← Previous: Race Conditions](04-race-conditions.md) | [Back to index](README.md) | [Next: SemaphoreSlim →](06-semaphore-slim.md)
