# Computer Architecture

> Read the questions, think about your answer, then click to reveal.

---

### 1. What are the stages of the fetch-decode-execute cycle?

<details>
<summary>Reveal answer</summary>

Every CPU core runs the same loop:

| Stage | What happens |
|-------|--------------|
| **Fetch** | Read the next instruction from memory (pointed to by the instruction pointer) |
| **Decode** | Interpret the bits — which operation, which operands |
| **Execute** | Run the operation (arithmetic, memory access, branch) |
| **Write-back** | Store the result in a register or memory |

A single "instruction" is primitive — an add, a load, a branch. A single C# statement compiles into many such instructions.

Deep dive: [CPU Internals](../04-computer-architecture/01-cpu-internals.md)

</details>

---

### 2. What is pipelining and why does it matter?

<details>
<summary>Reveal answer</summary>

Modern CPUs **overlap the stages** of consecutive instructions so that multiple instructions are in flight at once. With a full pipeline, the CPU retires about one instruction per cycle even though each instruction takes many cycles end-to-end. Real x86-64 pipelines are 15–20+ stages deep and superscalar (several pipelines running in parallel).

**Stalls** break the flow:
- **Data hazards:** an instruction depends on the result of the previous one
- **Memory stalls:** a cache miss to RAM costs hundreds of cycles
- **Branch mispredictions:** the CPU guessed wrong; the speculative work is discarded

Two CPUs at the same clock speed can have very different real performance because of IPC (instructions per cycle), which depends on pipeline depth and how often it stalls.

Deep dive: [CPU Internals](../04-computer-architecture/01-cpu-internals.md)

</details>

---

### 3. What is branch prediction and why does a sorted array run faster than a shuffled one?

<details>
<summary>Reveal answer</summary>

At every `if`, `while`, or conditional branch, the CPU must decide what to execute next before the condition is evaluated. Instead of stalling, it **predicts** the branch direction and speculatively executes that path. A correct prediction costs nothing; a misprediction flushes the pipeline and costs ~15–20 cycles on modern cores.

Branch predictors track history patterns, so **predictable** branches (always taken, always not taken, or following a simple pattern) are nearly free. **Unpredictable** branches in tight loops destroy performance.

The classic demo: `if (data[i] >= 128) sum += data[i]` inside a loop.

- Sorted array: the branch follows a clean pattern (false, false, ..., true, true). Near-100% prediction accuracy, no penalty.
- Shuffled array: roughly 50/50, so the predictor misses often. Same instruction count, several times slower.

Deep dive: [CPU Internals](../04-computer-architecture/01-cpu-internals.md)

</details>

---

### 4. What is the difference between a physical core, a logical processor, and a thread?

<details>
<summary>Reveal answer</summary>

Three layers that are often confused:

| Term | Definition |
|------|------------|
| **Physical processor (socket / CPU)** | The physical chip package. A machine can have one or more. |
| **Core** | One execution unit inside a physical processor |
| **Logical processor (hardware thread)** | One hardware context a core exposes to the OS. With SMT enabled, a core has more than one. |
| **OS thread** | A thread of execution managed by the kernel. Many OS threads can share the same logical processor over time. |
| **Managed thread (.NET)** | A `System.Threading.Thread`. The CLR maps it to an OS thread. |

Windows defines it precisely: "A *logical processor* is one logical computing engine from the perspective of the operating system, application or driver. A *core* is one processor unit, which can consist of one or more logical processors."

An "8-core / 16-thread" CPU has 8 physical cores, each with 2 logical processors via SMT — so the OS sees 16 logical processors.

Deep dive: [Cores and Threads](../04-computer-architecture/02-cores-and-threads.md)

</details>

---

### 5. What is SMT / Hyper-Threading and why is it not the same as doubling the cores?

<details>
<summary>Reveal answer</summary>

**SMT (Simultaneous Multi-Threading)** lets one physical core appear as two (or more) logical processors. Intel markets its implementation as **Hyper-Threading**; AMD calls it **SMT**.

Each logical processor gets its own architectural state (registers, instruction pointer), but the two **share** the execution units (ALU, FPU, SIMD), L1 and L2 caches, and decoders. The second logical processor fills in cycles when the first one stalls (cache miss, branch misprediction).

Two real cores can genuinely run two instructions at once. An SMT pair only can if the two threads need *different* execution resources at the same moment. Typical SMT speedup on multi-threaded workloads is 15–30%, not 100%.

> Interview trap: "My CPU has 8 cores and 16 threads" is marketing shorthand for 8 cores with SMT. "16 threads" here means hardware threads, not software threads.

Deep dive: [Cores and Threads](../04-computer-architecture/02-cores-and-threads.md)

</details>

---

### 6. What does Environment.ProcessorCount return in .NET?

<details>
<summary>Reveal answer</summary>

It returns the **number of logical processors available to the current process** — fixed at runtime startup. It's SMT-aware, so on an 8-core / 16-thread CPU it returns 16.

On modern .NET (Linux/macOS for all versions, Windows from .NET 6), it returns the **minimum** of:
- The number of logical processors on the machine
- The number of processors the process is affinitized to (if CPU affinity is set)
- The CPU utilization limit rounded up (if a limit is set)

This last point is what makes `ThreadPool` behave correctly inside containers: in a Kubernetes pod with a 2-CPU limit, `Environment.ProcessorCount` returns 2, not the 64 cores on the node.

**Source:** [Microsoft Learn — Environment.ProcessorCount](https://learn.microsoft.com/en-us/dotnet/api/system.environment.processorcount)

Deep dive: [Cores and Threads](../04-computer-architecture/02-cores-and-threads.md)

</details>

---

### 7. When should you use `new Thread(...)` vs the ThreadPool?

<details>
<summary>Reveal answer</summary>

| | `new Thread(...)` | `ThreadPool` / `Task.Run` |
|---|---|---|
| Creation cost | Expensive (allocates a new OS thread, ~1 MB stack) | Cheap (reuses a pool thread) |
| Lifetime | Dies when the delegate returns | Returned to the pool for reuse |
| Best for | Long-running, dedicated work with custom priority or stack size | Short-lived CPU or I/O work |

For almost everything in modern .NET, use `Task`, `async/await`, or `Parallel.*` — all of which use the ThreadPool. Only reach for a raw `Thread` when you specifically need a long-lived, dedicated OS thread.

Deep dive: [Cores and Threads](../04-computer-architecture/02-cores-and-threads.md)

</details>

---

### 8. Why is reading from RAM sometimes hundreds of times slower than reading from a register?

<details>
<summary>Reveal answer</summary>

The memory hierarchy:

| Level | Latency (order of magnitude) | Size |
|-------|------------------------------|------|
| Register | <1 ns | bytes |
| L1 cache (per core) | ~1 ns | ~32–64 KB |
| L2 cache (per core) | ~3–10 ns | ~256 KB–1 MB |
| L3 cache (shared) | ~10–30 ns | tens of MB |
| Main RAM | ~60–100 ns | GBs |
| SSD / Disk | microseconds to milliseconds | TBs |

Each level is about an order of magnitude slower than the one above. CPUs hide this by fetching data into caches speculatively and keeping hot data close to the core — but if your code accesses memory in a pattern the cache can't predict, it pays the RAM price on every access.

Deep dive: [Memory Hierarchy](../04-computer-architecture/03-memory-hierarchy.md)

</details>

---

### 9. Why does a `List<T>` usually beat a `LinkedList<T>` even for `O(n)` iteration?

<details>
<summary>Reveal answer</summary>

Both are `O(n)` in Big-O terms, but the constant factor is dominated by cache behavior.

- `List<T>` is backed by a **contiguous array**. Iterating it has excellent **spatial locality** — the CPU prefetches the next cache line, so most accesses hit L1.
- `LinkedList<T>` stores each node as a separate heap allocation, chained by references. Each node can be anywhere in memory, so iteration often misses the cache on every hop.

In practice, iterating a large array is often **10×+ faster** than iterating a linked list with the same element count, even though the algorithmic complexity is identical. This is a major reason `List<T>` is the default collection in .NET.

Deep dive: [Memory Hierarchy](../04-computer-architecture/03-memory-hierarchy.md)

</details>

---

### 10. What is false sharing?

<details>
<summary>Reveal answer</summary>

Caches don't fetch single bytes — they fetch **cache lines** (typically 64 bytes on x86-64). **False sharing** happens when two threads write to *different* variables that happen to sit on the *same* cache line. The cache coherence protocol treats it as contention: the line bounces between the cores' private caches on every write.

```csharp
class Counters
{
    public long A; // offset 0  — same 64-byte cache line as B
    public long B; // offset 8
}
```

If thread 1 writes `A` in a tight loop and thread 2 writes `B`, the threads *think* they're independent but the hardware is serializing them. Symptom in a profile: high cache-coherence traffic, poor scaling with added threads.

**Fix:** pad the fields onto separate cache lines (`[StructLayout(LayoutKind.Explicit)]` + `[FieldOffset(...)]`). Only worth doing when profiling shows it matters.

Deep dive: [Memory Hierarchy](../04-computer-architecture/03-memory-hierarchy.md)

</details>

---

### 11. Why is iterating an array of structs often faster than an array of class instances?

<details>
<summary>Reveal answer</summary>

An array of **structs** (`Point[]` where `Point` is a value type) stores the data contiguously in memory — iterating the array walks a straight line of cache lines, and the hardware prefetcher predicts the next read trivially.

An array of **class** instances (`Point[]` where `Point` is a reference type) stores only references in the array; the actual objects are scattered across the heap. Every element access is a **pointer chase** that can miss the cache and stall the CPU for dozens of cycles.

This is called "cache-unfriendly" access. Data-oriented design (structure of arrays, tight struct layouts) exploits the cache; class-heavy object graphs fight it.

Deep dive: [Memory Hierarchy](../04-computer-architecture/03-memory-hierarchy.md)

</details>

---

### 12. What is a branch mispredict, and how can it tank performance?

<details>
<summary>Reveal answer</summary>

Modern CPUs **speculatively execute** instructions past a branch (`if`, `while`, `switch`) based on the **branch predictor's** guess. If the guess is right, work is committed for free. If it's wrong, the pipeline is **flushed** and work started on the wrong path is discarded — typically a 10–20 cycle penalty per mispredict.

Well-predicted branches (same outcome repeatedly) are nearly free. Pathological cases (branching on random data) can double or triple the cost of a tight loop.

The classic demo: sorting an array *before* summing only the elements above a threshold runs faster than summing the unsorted array, because the sorted version's branch is predictable.

Deep dive: [CPU Internals](../04-computer-architecture/01-cpu-internals.md)

</details>

---

### 13. What is the difference between a physical core, a logical core, and a .NET thread?

<details>
<summary>Reveal answer</summary>

- **Physical core** — a real execution unit on the CPU die. Has its own execution resources (ALU, FPU) and a share of the cache.
- **Logical core** — what the OS sees. With **Simultaneous Multithreading** (Intel Hyper-Threading, AMD SMT), one physical core exposes 2 logical cores by interleaving instructions from two threads to hide memory-access stalls. Throughput gain is usually 15–30%, not 2×.
- **.NET thread** — a software construct. The OS scheduler maps runnable threads onto logical cores. There can be far more threads than cores; the scheduler time-slices them.

`Environment.ProcessorCount` returns **logical** cores — the number .NET uses to size its thread pool and `Parallel.ForEach` degree of parallelism by default.

Deep dive: [Cores and Threads](../04-computer-architecture/02-cores-and-threads.md)

</details>

---

[Back to index](README.md)
