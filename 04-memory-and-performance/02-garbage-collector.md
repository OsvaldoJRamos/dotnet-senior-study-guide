# Garbage Collector (GC)

## What it is

The **Garbage Collector (GC)** in .NET automatically reclaims memory that is no longer in use, freeing developers from manual memory management.

### What the GC does:
1. **Tracks object references** — monitors all objects on the heap and tracks which ones are still in use
2. **Reclaims unused memory** — when it detects that an object is no longer referenced, it marks it as "garbage" and reclaims its memory
3. **Improves performance** — runs periodically to keep memory usage optimized. It also reduces fragmentation by compacting the heap

## Generations

The GC uses a concept of **generations** to make the process more efficient.

### Generation 0 (Gen 0)
- This is where new objects are initially placed
- Objects in Gen 0 are short-lived (e.g., temporary variables)
- The GC checks Gen 0 frequently because most objects tend to become unnecessary quickly

### Generation 1 (Gen 1)
- If an object survives a Gen 0 collection (i.e., it is still in use), it is promoted to Gen 1
- Objects in Gen 1 are considered to have a medium lifetime
- The GC checks Gen 1 less frequently than Gen 0

### Generation 2 (Gen 2)
- Objects that survive a Gen 1 collection move to Gen 2
- These are long-lived objects (e.g., static data, large objects used throughout the application's lifetime)
- Gen 2 is collected less frequently because it is assumed these objects will remain for a long time

> The **Large Object Heap (LOH)** is a separate physical heap, not a generation; it is collected together with Gen 2 collections. Objects larger than ~85,000 bytes are allocated there directly.

### Why generations?

The reason for dividing memory into generations is to **optimize the collection process**. Short-lived objects (Gen 0) are collected frequently, while long-lived objects (Gen 2) are left alone unless absolutely necessary.

## Example

```csharp
class Program
{
    static void Main()
    {
        // Generation 0 - Short-lived objects
        for (int i = 0; i < 1000; i++)
        {
            var tempObject = new MyObject(); // Allocated in Gen 0
        }

        // Force garbage collection and print the generation of an object
        GC.Collect(); // Forces the garbage collection to run
        MyObject longLivedObject = new MyObject(); // Created in Gen 0

        Console.WriteLine($"Generation of longLivedObject: {GC.GetGeneration(longLivedObject)}");

        // Promote longLivedObject to Gen 1 by forcing another collection
        GC.Collect();
        Console.WriteLine($"Generation of longLivedObject after GC: {GC.GetGeneration(longLivedObject)}");

        // Promote to Gen 2
        GC.Collect();
        Console.WriteLine($"Generation of longLivedObject after second GC: {GC.GetGeneration(longLivedObject)}");
    }
}

class MyObject
{
    public MyObject()
    {
        // Simulate some memory allocation
        byte[] buffer = new byte[1024];
    }
}
```

> Avoid `GC.Collect()` in production code — forcing collections defeats the GC's adaptive heuristics and almost always makes performance worse. The calls above are for demonstration only.

## Best practices

1. **Avoid Memory Leaks** — always release unmanaged resources (such as database connections) using the `Dispose` method or `using` blocks
2. **Use IDisposable** — implement `IDisposable` for any class that handles unmanaged resources
3. **Be careful with large objects** — large objects (arrays, buffers, etc.) can end up in Generation 2 and stay there for a long time. Use them carefully

> Memory management in C# applications is generally automated thanks to the **Garbage Collector (GC)**, but this **does not mean developers are free from concerns**. There are fundamental practices to ensure efficiency and **avoid memory leaks**.

---

[← Previous: Stack and Heap](01-stack-and-heap.md) | [Back to index](README.md) | [Next: Memory Optimization →](03-memory-optimization.md)
