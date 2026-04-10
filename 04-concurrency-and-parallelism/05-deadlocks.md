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

### 3. Avoid `lock` on external resources (e.g., I/O, database)

Do not use `lock` in `async` methods, as it can block threads from the pool:

```csharp
// DO NOT do this
lock (locker)
{
    await database.SaveAsync(); // dangerous: can freeze
}
```

**Solution:** use asynchronous synchronization mechanisms (`SemaphoreSlim`).

### 4. Use the Task Parallel Library (TPL)

The TPL is a powerful concurrency framework in .NET that can help prevent deadlocks. It manages concurrency automatically and ensures that tasks do not interfere with each other.

```csharp
Task.Run(() =>
{
    // Access the resources
});

Task.Run(() =>
{
    // Access the resources
});
```

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
