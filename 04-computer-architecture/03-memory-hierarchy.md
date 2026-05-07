# Memory Hierarchy

Not all memory is equal. A CPU core can add two registers in a single cycle, but reading a value from main RAM can cost **hundreds** of cycles. To hide that gap, modern CPUs use a **hierarchy** of progressively larger, slower, cheaper memories. Understanding this hierarchy is what separates developers who write "correct code" from those who write code that actually runs fast.

## The hierarchy

```
┌─────────────────────────────┐    fastest, smallest, most expensive
│        Registers            │    <1 ns,      ~kilobytes
├─────────────────────────────┤
│     L1 cache (per core)     │    ~1 ns,      ~32–64 KB per core
├─────────────────────────────┤
│     L2 cache (per core)     │    ~3–10 ns,   ~256 KB–1 MB per core
├─────────────────────────────┤
│     L3 cache (shared)       │    ~10–30 ns,  tens of MB
├─────────────────────────────┤
│         Main RAM            │    ~60–100 ns, GBs
├─────────────────────────────┤
│         SSD / Disk          │    microseconds to milliseconds
└─────────────────────────────┘    slowest, largest, cheapest
```

The numbers above are orders of magnitude, not precise values — they vary per CPU model. What you should remember is the **shape**: each level is roughly an order of magnitude slower than the one above it.

> A modern rule of thumb (popularized by Jeff Dean's latency numbers and Peter Norvig's "Teach Yourself Programming in Ten Years"): a cache hit is measured in nanoseconds, a cache miss to RAM is measured in tens of nanoseconds, and a disk seek is measured in milliseconds — a factor of millions.

## Cache lines

Caches don't fetch single bytes. They fetch fixed-size blocks called **cache lines**. On mainstream x86-64 CPUs (Intel and AMD), a cache line is **64 bytes**. ARMv8 cores vary but 64 bytes is also common.

When your code reads `array[i]`, the CPU actually loads the 64-byte block containing `array[i]` into L1. If you then read `array[i+1]`, it's already there — essentially free. This is why **contiguous memory access is fast**.

## Locality of reference

Good cache behavior depends on two kinds of locality:

| Type | Definition | Example |
|------|------------|---------|
| **Temporal locality** | Recently accessed data is likely to be accessed again soon | A counter updated every iteration of a loop |
| **Spatial locality** | Data near recently accessed data is likely to be accessed soon | Iterating sequentially through an array |

Contiguous structures (arrays, `Span<T>`, `List<T>`) have excellent spatial locality. Linked structures (`LinkedList<T>`, trees with pointer hops) do not — each node can be anywhere in memory, so traversal tends to miss the cache on every node.

### Example: array vs linked list

```csharp
// Fast — contiguous, cache-friendly
int[] arr = new int[1_000_000];
int sum = 0;
for (int i = 0; i < arr.Length; i++) sum += arr[i];

// Slow — every node can be anywhere on the heap
var list = new LinkedList<int>();
// ... populate with 1,000,000 items
int sum2 = 0;
foreach (var v in list) sum2 += v;
```

Despite the same Big-O complexity (`O(n)`), the array version often runs **10× or more** faster on modern hardware purely because of cache behavior. This is one of the reasons `List<T>` (backed by an array) is the default list in .NET, not `LinkedList<T>`.

## Cache coherence and false sharing

When you have multiple cores, each with its own L1 and L2, the hardware must keep them consistent. If core 0 writes a cache line that core 1 has cached, core 1's copy must be invalidated. This is **cache coherence** and it's handled transparently by protocols like MESI (Modified, Exclusive, Shared, Invalid).

The subtle performance trap is **false sharing**: two threads writing to *different* variables that happen to sit on the **same cache line** force the coherence protocol to keep bouncing that line between their caches.

```csharp
class Counters
{
    public long A; // offset 0
    public long B; // offset 8 — same 64-byte cache line as A
}
```

If thread 1 writes `A` in a tight loop and thread 2 writes `B` in a tight loop, the cache line holding both bounces back and forth. The threads *think* they're independent, but the hardware treats them as contending.

.NET provides `[StructLayout(LayoutKind.Explicit)]` with `[FieldOffset(...)]` and padding to force fields onto separate cache lines when false sharing matters. Most of the time you don't need this — but recognizing the symptom in a profile is a senior-level skill.

## Virtual memory (very briefly)

The addresses your program uses are **virtual addresses**, translated to physical RAM addresses by the CPU's MMU using page tables. Translations are cached in the **TLB (Translation Lookaside Buffer)** — a TLB miss is another kind of stall worth knowing exists.

This is also what makes the OS able to:

- Give each process the illusion of a private, contiguous address space
- Swap pages out to disk when RAM is full (paging)
- Share pages (e.g., the same DLL mapped into multiple processes)

## Why this matters for .NET developers

Everything in this file is the "why" behind several .NET performance patterns you'll see later:

| Pattern | Why it's fast |
|---------|---------------|
| Prefer `List<T>` over `LinkedList<T>` | Contiguous memory, spatial locality |
| `Span<T>` / `Memory<T>` over sliced arrays | Avoids copying; keeps data in contiguous regions |
| `StringBuilder` over repeated string concatenation | Avoids allocating many small strings scattered on the heap |
| Struct arrays (`T[]` where `T` is a struct) | Keeps data inline, not pointer-chased |
| Pool allocations (`ArrayPool<T>.Shared`) | Reuses memory that is likely still hot in cache |

All of these come back in [Memory and Performance](../05-memory-and-performance/README.md), where we cover the GC, LOH, memory optimization, and struct vs class tradeoffs.

---

[← Previous: Cores and Threads](02-cores-and-threads.md) | [Back to index](README.md)
