# CPU Internals

The **CPU (Central Processing Unit)** is the component that actually executes your code. Every line of C# you write eventually becomes a sequence of machine instructions that a CPU core runs. Understanding how the CPU works explains *why* certain patterns are fast and others are slow — and it's the foundation for memory, concurrency, and performance topics later in the guide.

## The fetch-decode-execute cycle

At the lowest level, every CPU core repeats the same loop:

| Stage | What happens |
|-------|--------------|
| **Fetch** | Read the next instruction from memory (pointed to by the program counter / instruction pointer) |
| **Decode** | Interpret the bits of the instruction — which operation, which operands |
| **Execute** | Run the operation (arithmetic, memory read/write, branch, etc.) |
| **Write-back** | Store the result in a register or memory |

A single "instruction" is a primitive like "add two integers" or "load a value from memory into a register" — far smaller than a C# statement.

## Main components

```
        ┌──────────────────────────────────────┐
        │            CPU Core                  │
        │                                      │
        │   ┌───────┐   ┌─────┐   ┌─────────┐  │
        │   │Control│──▶│ ALU │   │Registers│  │
        │   │ Unit  │   └─────┘   └─────────┘  │
        │   └───────┘      ▲           ▲       │
        │       │          │           │       │
        │       └──────────┴───────────┘       │
        │              L1 / L2 cache           │
        └──────────────────────────────────────┘
                           │
                         L3 / RAM
```

- **Control Unit (CU):** orchestrates fetch, decode, and dispatches work to the ALU
- **ALU (Arithmetic Logic Unit):** performs integer/bitwise operations. Floating-point is typically handled by a dedicated **FPU** (or SIMD units like SSE/AVX)
- **Registers:** a small, extremely fast set of storage slots directly inside the CPU (e.g., on x86-64: `RAX`, `RBX`, `RIP` for the instruction pointer, `RSP` for the stack pointer)
- **Cache (L1/L2/L3):** hierarchy of small, fast memories between registers and RAM — covered in [Memory Hierarchy](03-memory-hierarchy.md)

## Clock speed vs. work done

**Clock speed** (GHz) is how many cycles the CPU runs per second. A 3 GHz CPU runs 3 billion cycles per second. But *how much work* happens per cycle depends on:

- **IPC (Instructions Per Cycle):** modern cores execute *multiple* instructions per cycle via pipelining and superscalar execution
- **Pipeline depth and how often it stalls**
- **Cache hits vs misses** (a miss to main RAM can cost hundreds of cycles)

> Two CPUs at the same clock speed can have very different real-world performance.

## Pipelining

Instead of finishing one instruction before fetching the next, modern CPUs **overlap the stages** of consecutive instructions. A simplified 5-stage pipeline:

```
Cycle:        1    2    3    4    5    6    7
Instr 1:     F -> D -> E -> M -> W
Instr 2:          F -> D -> E -> M -> W
Instr 3:               F -> D -> E -> M -> W
```

With a full pipeline, the CPU effectively retires one instruction per cycle even though each instruction takes 5 cycles end-to-end. Real x86-64 pipelines are much deeper (15–20+ stages) and **superscalar** (multiple pipelines running in parallel).

### Pipeline stalls

Anything that breaks the steady flow creates a **stall** (also called a bubble):

- **Data hazards:** an instruction needs the result of the previous one
- **Memory stalls:** waiting for a cache miss to resolve (hundreds of cycles if it hits RAM)
- **Branch mispredictions:** see below

> Code that is friendly to the pipeline and the cache runs dramatically faster than code that isn't — even if both compile to "the same" instruction count.

## Branch prediction

Whenever the CPU hits an `if`, a `while`, or any conditional branch, it doesn't know which path to take until the condition is evaluated. Waiting would stall the pipeline. Instead, the CPU **predicts** which way the branch will go and speculatively executes that path.

- **Correct prediction:** no penalty, pipeline keeps flowing
- **Misprediction:** the speculative work is discarded and the pipeline is flushed — a penalty of ~15–20 cycles on modern cores

Branch predictors are sophisticated (they track history patterns), so predictable branches — always taken, always not taken, or following a simple pattern — are nearly free. Unpredictable branches, especially inside tight loops, are where performance dies.

### Interview example: the famous "sorted array is faster" case

```csharp
// Sum only elements >= 128
int sum = 0;
for (int i = 0; i < data.Length; i++)
{
    if (data[i] >= 128) sum += data[i];
}
```

If `data` is **sorted**, the branch follows a predictable pattern (always false, then always true) and the predictor wins every time. If `data` is **random**, the predictor misses roughly 50% of the time, and the same code can run several times slower — with identical instruction count.

> This is a classic Stack Overflow question and a common interview demo that branch prediction is real, measurable, and a language-agnostic property of the CPU.

## Speculative execution and security

Speculative execution is not just guessing a branch — the CPU can execute *any* future instructions whose results can be discarded if not needed. This is a huge performance win, but it also leaked data via side channels in the **Spectre** and **Meltdown** vulnerabilities (2018): an attacker could observe cache-timing side effects of instructions that were speculatively executed and then discarded.

> You don't need to know Spectre/Meltdown in depth for most interviews, but knowing that speculative execution exists and *why* it was risky shows you understand how modern CPUs actually work.

## Why this matters for .NET developers

The .NET JIT and runtime already optimize for all of the above, but your code still affects how well the CPU can:

- **Keep the pipeline full** — avoid tight loops full of unpredictable branches
- **Hit the cache** — prefer contiguous arrays (`Span<T>`, `List<T>`) over linked structures
- **Vectorize** — use `Span<T>`, `System.Numerics.Vector<T>`, and `System.Runtime.Intrinsics` when you need SIMD

All of these come back in [Memory Hierarchy](03-memory-hierarchy.md) and the [Memory and Performance](../04-memory-and-performance/README.md) section.

---

[Back to index](README.md) | [Next: Cores and Threads →](02-cores-and-threads.md)
