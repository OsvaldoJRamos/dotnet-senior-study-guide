# 05 - Memory and Performance

## Content

1. [Stack and Heap](01-stack-and-heap.md) - Where data is stored in memory
2. [Garbage Collector](02-garbage-collector.md) - Generations, SOH/LOH/POH, Workstation vs Server, Background GC, region-based heap, DATAS
3. [Memory Optimization](03-memory-optimization.md) - StringBuilder, Span/Memory, stackalloc, ArrayPool, BinaryPrimitives
4. [Memory Leak](04-memory-leak.md) - Events, timers, AsyncLocal capture, WeakReference, ConditionalWeakTable
5. [Structs vs Classes](05-structs-vs-classes.md) - Value vs reference types, `readonly struct`, `in`/`ref readonly`, `ref struct`
6. [Async and Memory](06-async-and-memory.md) - Async state machine allocations, Task vs ValueTask, Memory<T> across await
7. [JIT and Runtime](07-jit-and-runtime.md) - Tiered compilation, Dynamic PGO, devirtualization, escape analysis
8. [Diagnostics](08-diagnostics.md) - BenchmarkDotNet, dotnet-counters/trace/dump/gcdump, PerfView, dotMemory
9. [System.IO.Pipelines](09-pipelines.md) - PipeReader/PipeWriter, ReadOnlySequence<T>, IBufferWriter<T>
10. [Memory Model](10-memory-model.md) - Visibility vs ordering, acquire-release, lock-free patterns, double-checked locking
11. [Modern String APIs](11-modern-string-apis.md) - SearchValues<T>, string.Create, DefaultInterpolatedStringHandler, u8 literals

---

[Back to index](../README.md)
