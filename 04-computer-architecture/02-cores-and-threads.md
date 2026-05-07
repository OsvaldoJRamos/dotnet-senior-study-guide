# Cores and Threads

This topic is the source of constant confusion because the word "thread" gets used at three different layers: **hardware thread**, **OS thread**, and **managed (.NET) thread**. An interviewer will often ask you to untangle them.

## The three definitions Windows uses

Microsoft's Windows documentation defines the hardware terms precisely:

> "A *logical processor* is one logical computing engine from the perspective of the operating system, application or driver. A *core* is one processor unit, which can consist of one or more logical processors. A *physical processor* can consist of one or more cores. A physical processor is the same as a processor package, a socket, or a CPU."
>
> — [Microsoft Learn — Processor Groups](https://learn.microsoft.com/en-us/windows/win32/procthread/processor-groups)

So the containment goes:

```
Socket (physical processor)
└── Core (one execution unit)
    └── Logical processor (what the OS schedules onto)
```

A modern 8-core CPU with 2 logical processors per core reports **16 logical processors** to the OS.

## Why one core can have multiple logical processors: SMT

**SMT (Simultaneous Multi-Threading)** is the hardware feature that lets a single physical core appear as multiple logical processors. Intel markets its implementation as **Hyper-Threading**; AMD calls it **SMT**. Most mainstream implementations provide **2 logical processors per core**.

Inside one SMT-enabled core:

| Duplicated per logical processor | Shared between logical processors |
|----------------------------------|-----------------------------------|
| Architectural registers (so each thread has its own "CPU state") | ALU, FPU, SIMD units |
| Instruction pointer | L1 / L2 cache |
| Interrupt state | Decoders, execution ports |

The goal is to keep the core's execution units busy: when one thread stalls (cache miss, branch misprediction), the other thread can use the freed-up units.

**SMT is not the same as "two cores."** Two real cores can genuinely run two instructions at once. An SMT pair can only do so if the two threads need *different* execution resources at the same moment. Measured SMT speedups on multi-threaded workloads are usually in the tens of percent (commonly cited ranges are 15–30%), not 100% — the exact number varies widely per workload and microarchitecture.

> Interview trap: "My CPU has 8 cores and 16 threads." What they mean is 8 physical cores with SMT, giving 16 logical processors. "16 threads" here is a marketing shorthand for hardware threads, not software threads.

## Hardware thread vs. OS thread vs. managed thread

| Layer | What it is |
|-------|------------|
| **Hardware thread (logical processor)** | A hardware context on a core. How many you have is fixed by the CPU. |
| **OS thread** | A thread of execution managed by the operating system kernel. Created via `CreateThread` on Windows, `pthread_create` on Linux. Has its own stack and register state. |
| **Managed thread (.NET)** | A `System.Threading.Thread` instance. The CLR maps it to an OS thread. |

You can have **thousands of OS threads** on a machine with 16 logical processors — the OS just keeps swapping them in and out. Each logical processor runs exactly one OS thread at any given instant; the rest are suspended.

### OS scheduling

The OS scheduler decides which OS threads run on which logical processors and when to switch. Key points:

- A **context switch** saves the registers and stack pointer of the outgoing thread, restores the incoming thread's state, and often invalidates caches. It's not free — typically 1–10 microseconds plus cache pollution.
- Threads have **priorities**, but the scheduler still aims for fairness
- Threads can have a **processor affinity** (pinned to specific logical processors) — .NET exposes this via `Process.ProcessorAffinity`

## .NET mapping

In .NET, you rarely create raw `Thread` instances directly. Most concurrency goes through the **ThreadPool**, which manages a pool of worker threads reused across many short-lived work items. `Task.Run`, `Parallel.ForEach`, `async/await` continuations — all default to the ThreadPool.

### `Environment.ProcessorCount`

The standard way to ask "how many logical processors do I have?" from .NET:

```csharp
int logicalProcessors = Environment.ProcessorCount;
Console.WriteLine($"Logical processors: {logicalProcessors}");
```

What the docs say it actually returns (on modern .NET):

> "Gets the number of processors available to the current process. [...] On Linux and macOS systems for all .NET versions and on Windows systems starting with .NET 6, this API returns the minimum of:
> - The number of logical processors on the machine.
> - If the process is running with CPU affinity, the number of processors that the process is affinitized to.
> - If the process is running with a CPU utilization limit, the CPU utilization limit rounded up to the next whole number.
>
> The value returned by this API is fixed at .NET runtime startup for the process lifetime."
>
> — [Microsoft Learn — Environment.ProcessorCount](https://learn.microsoft.com/en-us/dotnet/api/system.environment.processorcount)

Two practical consequences:

- It counts **logical** processors (SMT-aware), not physical cores. On an 8-core / 16-thread CPU, it returns 16.
- It respects containers and cgroups CPU limits. In a container limited to 2 CPUs, it returns 2 — this is what allows `ThreadPool` defaults to behave sensibly in Kubernetes.

### `Thread` vs the ThreadPool

| | `new Thread(...)` | `ThreadPool` / `Task.Run` |
|---|---|---|
| Creation cost | Expensive (allocates a new OS thread) | Cheap (reuses pool threads) |
| Default stack size | 1 MB on Windows | 1 MB on Windows |
| Lifetime | Dies when the delegate returns | Returned to the pool for reuse |
| Best for | Long-running, dedicated work (e.g., a custom scheduler) | Short-lived CPU or I/O work |

> For almost everything in modern .NET, you want `Task` / `async` / the ThreadPool. Spin up a raw `Thread` only when you specifically need a long-lived, dedicated OS thread with its own priority or stack size.

## How many threads should I use?

The short answers:

- **CPU-bound work:** roughly `Environment.ProcessorCount` concurrent threads. More just causes context switching.
- **I/O-bound work:** use `async/await` instead of extra threads — blocked threads still occupy memory and the ThreadPool.
- **Mixed workloads:** let the ThreadPool figure it out. The docs describe it as creating and destroying worker threads "in order to optimize throughput, which is defined as the number of tasks that complete per unit of time" ([MS Learn — The managed thread pool](https://learn.microsoft.com/en-us/dotnet/standard/threading/the-managed-thread-pool)).

This is the bridge to the [Concurrency and Parallelism](../06-concurrency-and-parallelism/README.md) section, which goes deep on how to *use* these threads correctly — locks, races, deadlocks, async/await.

---

[← Previous: CPU Internals](01-cpu-internals.md) | [Next: Memory Hierarchy →](03-memory-hierarchy.md) | [Back to index](README.md)
