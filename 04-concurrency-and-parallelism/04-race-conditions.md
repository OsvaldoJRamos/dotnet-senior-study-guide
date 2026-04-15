# Race Conditions

## What are they

**Race conditions** occur when multiple threads access the same shared resource, leading to unexpected behavior due to the lack of synchronization between threads. For example, two threads executing addClick and removeClick at the same time. The total number of clicks may differ depending on which thread executes first.

## How to avoid them

### 1. Locks

A synchronization technique to restrict access to a shared resource. An object that can only be held by one thread at a time. The thread must acquire the lock before accessing the resource and release it when done.

```csharp
object lockObj = new object();
int sharedVar = 0;

// Thread 1
lock (lockObj)
{
    sharedVar++;
}

// Thread 2
lock (lockObj)
{
    sharedVar++;
}
```

Using locks, only one thread can access `sharedVar` at a time, preventing race conditions.

### 2. Interlocked

Another way to avoid race conditions is by using the `Interlocked` class, which provides atomic operations on shared variables. Atomic operations are executed in a single step, without interruption from other threads.

```csharp
int sharedVar = 0;

// Thread 1
Interlocked.Increment(ref sharedVar);

// Thread 2
Interlocked.Increment(ref sharedVar);
```

`Interlocked.Increment` ensures that the increment operation is executed atomically, preventing race conditions.

> **Important:** `Interlocked` provides **atomicity for a single variable operation** (increment, exchange, compare-and-swap). It does **NOT** protect **composite invariants** across multiple fields. If your invariant spans more than one variable (e.g., "deduct from account A **and** credit account B"), you still need `lock` or `SemaphoreSlim`. Using `Interlocked` on each field independently can leave the object in an inconsistent intermediate state visible to other threads.

### 3. Thread Safety

When writing C# code, it is important to consider thread safety in the design of classes and methods. Thread-safe code can be accessed by multiple threads simultaneously without causing race conditions. You can ensure thread safety by using synchronization techniques such as locks or Interlocked operations, or by using immutable objects.

```csharp
public class ThreadSafeCounter
{
    private int count;
    private readonly object lockObj = new object();

    public int Increment()
    {
        lock (lockObj)
        {
            return ++count;
        }
    }

    public int Decrement()
    {
        lock (lockObj)
        {
            return --count;
        }
    }
}
```

## Thread-safe collections

.NET provides thread-safe collections in the `System.Collections.Concurrent` namespace:

```csharp
// Instead of List<T> + lock:
var bag = new ConcurrentBag<string>();
var dict = new ConcurrentDictionary<string, int>();
var queue = new ConcurrentQueue<string>();
```

---

[← Previous: Task, async/await](03-task-async-await.md) | [Back to index](README.md) | [Next: Deadlocks →](05-deadlocks.md)
