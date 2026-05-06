# 05 - Concurrency and Parallelism

## Content

1. [Parallelism vs Concurrency](01-parallelism-vs-concurrency.md) - Concepts and differences
2. [Parallel.ForEach and Parallel.Invoke](02-parallel-foreach-invoke.md) - Parallel task execution
3. [Task, async/await and PLINQ](03-task-async-await.md) - Asynchronous programming
4. [Race Conditions](04-race-conditions.md) - Locks and Interlocked
5. [Deadlocks](05-deadlocks.md) - Causes and prevention
6. [SemaphoreSlim](06-semaphore-slim.md) - Asynchronous concurrency control
7. [Locks in Depth](07-locks-in-depth.md) - `lock`, `Monitor`, `System.Threading.Lock`, `ReaderWriterLockSlim`, `Mutex`, `SpinLock`, `Interlocked`, `Volatile`
8. [The Thread Pool](08-thread-pool.md) - Worker vs IOCP threads, hill climbing, work stealing, starvation
9. [Channels](09-channels.md) - Async producer/consumer, bounded vs unbounded, backpressure
10. [Concurrent Collections](10-concurrent-collections.md) - Lock striping, lock-free, `ConcurrentDictionary`/`Queue`/`Stack`/`Bag`, immutables
11. [Task Lifecycle](11-task-lifecycle.md) - `TaskStatus` enum (8 states), `IsCompleted` vs `IsCompletedSuccessfully`, `await` vs `.Wait()` exception unwrapping, cancellation rule

## Useful Links

- [Concurrency vs Parallelism (Oxylabs)](https://oxylabs.io/blog/concurrency-vs-parallelism) — implementation-focused walkthrough with concrete benchmarks (Python examples, but the concepts apply broadly).

---

[Back to index](../README.md)
