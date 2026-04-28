# Garbage Collector (GC)

## What it is

The **Garbage Collector (GC)** in .NET automatically reclaims memory that is no longer in use, freeing developers from manual memory management.

### What the GC does:
1. **Tracks object references** — monitors all objects on the heap and tracks which ones are still in use
2. **Reclaims unused memory** — when it detects that an object is no longer referenced, it marks it as "garbage" and reclaims its memory
3. **Improves performance** — runs periodically to keep memory usage optimized. It also reduces fragmentation by compacting the heap

> Allocation in the managed heap is fast because the GC keeps a *bump pointer* to the next free address. Allocating an object is roughly: move the pointer N bytes, zero the memory, return the address. The cost lives in the **collection**, not in the allocation.

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

> Gen 2 collections are also called **full GCs** — they walk the entire managed heap. This is why "objects that survive to Gen 2" is the costly outcome to avoid.

### Why generations?

The reason for dividing memory into generations is to **optimize the collection process**. Short-lived objects (Gen 0) are collected frequently, while long-lived objects (Gen 2) are left alone unless absolutely necessary.

## SOH, LOH, and POH

The managed heap is logically split into multiple physical heaps:

| Heap | What goes there | Compaction |
|---|---|---|
| **Small Object Heap (SOH)** | Objects < 85,000 bytes (default LOH threshold) | Compacted normally |
| **Large Object Heap (LOH)** | Objects ≥ 85,000 bytes (typically large arrays) | Not compacted by default — generates fragmentation |
| **Pinned Object Heap (POH)** | Objects allocated via `GC.AllocateArray<T>(length, pinned: true)` (.NET 5+) | Not compacted (pinned by design) |

LOH and POH are physically separate from SOH but **logically collected together with Gen 2**.

### LOH details

Per Microsoft's GC fundamentals doc, the LOH "contains objects that are 85,000 bytes and larger, which are usually arrays. It's rare for an instance object to be extra large." Objects on the LOH are collected during Gen 2 collections; LOH is sometimes called *generation 3* in docs.

You can adjust the threshold via `System.GC.LOHThreshold` (`.NET Core 3.0`+) — the value must be **larger** than the default 85,000 bytes:

```json
{ "configProperties": { "System.GC.LOHThreshold": 120000 } }
```

You can also force LOH compaction once via `GCSettings.LargeObjectHeapCompactionMode` — but this is expensive; use sparingly.

### POH (.NET 5+)

When native code (P/Invoke) needs an array, the GC must not move it. Historically, this meant pinning (`fixed` / `GCHandle.Pinned`) inside the regular heap, which **disrupts compaction**. The POH solves this by giving pinned objects their own heap.

```csharp
byte[] pinnedBuffer = GC.AllocateArray<byte>(4096, pinned: true);
// Can be passed to native code without further pinning
```

Per the official `GC.AllocateArray<T>` docs, in **.NET 7 and earlier versions**: when `pinned` is `true`, `T` must not be a reference type or a type that contains object references. The restriction is design-level — POH objects are not scanned for cross-generation references because by definition they cannot hold them.

## GC flavors: Workstation vs Server

The two main flavors:

| Flavor | When to use | Default |
|---|---|---|
| **Workstation GC** | Single-threaded GC, smaller heap, lower pause overhead — for client apps | Default in raw runtime |
| **Server GC** | One GC thread and one heap **per logical processor**, higher throughput, larger heap | Off by default in raw runtime; **on by default in ASP.NET Core templates** via `<ServerGarbageCollection>true</ServerGarbageCollection>` |

Configuration:

```xml
<PropertyGroup>
  <ServerGarbageCollection>true</ServerGarbageCollection>
</PropertyGroup>
```

Or `System.GC.Server` in `runtimeconfig.json`, or `DOTNET_gcServer=1` env var (`.NET 6`+; older versions used `COMPlus_gcServer`).

> Server GC requires 2+ logical processors. On single-CPU hardware, the runtime falls back to Workstation regardless of configuration.

### Background GC (concurrent)

Orthogonal to the Workstation/Server choice. Background GC does most of the Gen 2 collection work **in parallel with the application threads**, with only short pauses at the start and end. Per the official docs:

- **Default: enabled** (`System.GC.Concurrent` defaults to `true`)
- Gen 0 and Gen 1 collections are still stop-the-world (they're already short)
- Gen 2 (full) is the one that benefits from being concurrent

Disable with `<ConcurrentGarbageCollection>false</ConcurrentGarbageCollection>` only if you know why — for example, when using `GC.RegisterForFullGCNotification`, which only works with non-concurrent GC.

## Region-based heap (.NET 7+)

Before .NET 7, the GC organized the heap into a few large *segments* (hundreds of MB each). Growing the heap meant allocating a new segment; shrinking meant releasing whole segments back to the OS — coarse and slow.

Starting in .NET 7, the GC reorganized into **regions** for 64-bit Windows and Linux:

- **SOH region size: 4 MB by default** (configurable via `System.GC.RegionSize` in .NET 10, or `DOTNET_GCRegionSize` env var since .NET 7)
- **UOH (LOH and POH) region size: 8× SOH = 32 MB by default**
- Regions are allocated on demand and can be re-purposed between generations

This gives the GC much finer granularity for growing, shrinking, and returning memory to the OS — and is the infrastructure that made DATAS possible.

## DATAS — Dynamic Adaptation To Application Sizes

A change in how Server GC sizes its heap. Verified history (per official docs):

- **Introduced in .NET 8** as opt-in (`DOTNET_GCDynamicAdaptationMode=1`)
- **Enabled by default starting in .NET 9** (per official runtime config docs)

### The motivation

Classic Server GC sized the heap based on throughput and core count. On a 48-core machine, the heap could grow large even with a light workload — costly in containers where memory has a price tag.

DATAS adapts heap size to **actual work**, not machine specs. Bursty workload → grows. Light → shrinks and returns memory. Per the docs, it uses a **Throughput Cost Percentage (TCP)** target (default 2%) that combines GC pause time and allocation wait time.

### Configuration

```json
{ "configProperties": { "System.GC.DynamicAdaptationMode": 1 } }
```

Or env var: `DOTNET_GCDynamicAdaptationMode=1`. Set to `0` to disable. The MSBuild property is `GarbageCollectionAdaptationMode`.

### When to disable

If your app is hot from the first request and cannot tolerate the heap-growth ramp (DATAS starts with one heap and grows on demand). For most server workloads — especially in containers — DATAS is a clear win. Always benchmark before disabling.

## Configuration cheat sheet

Common settings (verified against MS Learn `garbage-collector` runtime config docs):

| Setting | runtimeconfig.json | Env var |
|---|---|---|
| Server GC | `System.GC.Server` | `DOTNET_gcServer` |
| Background GC | `System.GC.Concurrent` | `DOTNET_gcConcurrent` |
| LOH threshold | `System.GC.LOHThreshold` | `DOTNET_GCLOHThreshold` |
| Heap hard limit (bytes) | `System.GC.HeapHardLimit` | `DOTNET_GCHeapHardLimit` |
| Heap hard limit (% of host) | `System.GC.HeapHardLimitPercent` | `DOTNET_GCHeapHardLimitPercent` |
| DATAS | `System.GC.DynamicAdaptationMode` | `DOTNET_GCDynamicAdaptationMode` |
| Region size (SOH) | `System.GC.RegionSize` (.NET 10) | `DOTNET_GCRegionSize` (.NET 7+) |

Per the docs, in containers the default `HeapHardLimitPercent` is **75%** of the container's memory limit when no explicit limit is set.

> Avoid `GC.Collect()` in production code — forcing collections defeats the GC's adaptive heuristics and almost always makes performance worse. Reserve it for tests and microbenchmarks where you need a clean baseline.

## Best practices

1. **Avoid memory leaks** — release unmanaged resources via `Dispose` / `using`
2. **Implement `IDisposable`** for any class that holds unmanaged resources
3. **Be careful with large objects** — they go straight to LOH (and Gen 2) and resist eviction
4. **Pool large/frequent buffers** with `ArrayPool<T>` to dodge LOH churn
5. **Keep allocations short-lived** — Gen 0 collection is essentially free; Gen 2 is not

> Memory management in C# applications is generally automated thanks to the **Garbage Collector (GC)**, but this **does not mean developers are free from concerns**. There are fundamental practices to ensure efficiency and **avoid memory leaks**.

---

[← Previous: Stack and Heap](01-stack-and-heap.md) | [Back to index](README.md) | [Next: Memory Optimization →](03-memory-optimization.md)
