# Diagnostics: Measuring Memory and Performance

The cardinal rule: **measure before you optimize**. Without a profile, every "performance fix" is guesswork. The .NET ecosystem ships excellent free tools — knowing which one to reach for is half the battle.

## BenchmarkDotNet — microbenchmarks

The standard for .NET microbenchmarks. Handles warm-up, multiple iterations, statistical analysis, and outlier detection. The most common misuse — running benchmarks under a debugger or in a `Stopwatch` loop — is exactly what BenchmarkDotNet exists to prevent.

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]
public class StringBench
{
    private readonly string _input = new string('x', 1024);

    [Benchmark(Baseline = true)]
    public string OldWay() => _input.Substring(10, 100);

    [Benchmark]
    public string NewWay() => _input.AsSpan(10, 100).ToString();
}

// Program.cs
BenchmarkRunner.Run<StringBench>();
```

Required attributes for any serious benchmark:

| Attribute | What it does |
|---|---|
| `[MemoryDiagnoser]` | Reports allocations per iteration — essential for any "did this reduce GC pressure?" comparison |
| `[Benchmark(Baseline = true)]` | Marks one method as the baseline; results show ratios |
| `[Params(...)]` | Runs the benchmark across multiple input sizes |
| `[GlobalSetup]` | One-time setup before the benchmark suite |

### Sample output

```
| Method  | Mean      | Allocated |
|-------- |---------- |---------- |
| OldWay  | 152.3 ns  | 240 B     |
| NewWay  |  87.5 ns  | 120 B     |
```

`Allocated: -` (zero) is the gold sign for memory-sensitive benchmarks.

### Running benchmarks correctly

The BenchmarkDotNet good-practices guide is explicit:

- Build in **Release** configuration. *"Never use the Debug build for benchmarking. Never. The debug version of the target method can run 10–100 times slower."*
- Don't run with a debugger attached. *"Never use an attached debugger (e.g. Visual Studio or WinDbg) during the benchmarking."*
- *"Turn off all of the applications except the benchmark process and the standard OS processes."*
- For laptops, *"keep it plugged in and use the maximum performance mode."*
- For meaningful comparisons, run the same benchmark suite on the same hardware.

## `dotnet-counters` — live runtime metrics

A console tool that taps into the runtime's `EventCounter` infrastructure. No code changes required — point it at a running process and watch.

```bash
dotnet tool install -g dotnet-counters

# Find your process
dotnet-counters ps

# Stream the System.Runtime counters (default)
dotnet-counters monitor -p <PID>

# Or specific counter providers
dotnet-counters monitor -p <PID> --counters System.Runtime,Microsoft.AspNetCore.Hosting
```

Useful counters for memory / GC investigation:

- `# of Gen 0 / Gen 1 / Gen 2 Collections` — frequencies; rising Gen 2 is a red flag
- `Allocation Rate (B/sec)` — how much you're allocating
- `GC Heap Size` — managed heap size
- `Working Set` — process RAM as seen by the OS
- `% Time in GC` — proportion of CPU spent in GC
- `ThreadPool Queue Length` — request backlog; high values mean thread starvation

`dotnet-counters` is the right tool for "something feels wrong in production, what's happening *right now*?".

## `dotnet-trace` — CPU and allocation traces

Captures a runtime trace for offline analysis. Like `perf record` for .NET.

```bash
dotnet tool install -g dotnet-trace

# Default profile: CPU sampling for 60s
dotnet-trace collect -p <PID> --duration 00:01:00

# Allocation tracking (verbose)
dotnet-trace collect -p <PID> --providers Microsoft-DotNETCore-SampleProfiler,Microsoft-Windows-DotNETRuntime:0x1:5
```

Analysis: open the resulting `.nettrace` file in **PerfView** (Windows) or **Speedscope** (cross-platform). Both produce flame graphs and allocation breakdowns.

The right tool for: "throughput is lower than expected; where is CPU actually going?"

## `dotnet-dump` and `dotnet-gcdump` — post-mortem analysis

`dotnet-dump` captures a **full process dump** (everything in memory, including unmanaged); `dotnet-gcdump` captures **just the managed GC heap** — much smaller and faster.

```bash
dotnet tool install -g dotnet-dump
dotnet tool install -g dotnet-gcdump

# Full dump (large file, full state)
dotnet-dump collect -p <PID>

# GC heap only (small, focused on managed memory)
dotnet-gcdump collect -p <PID>
```

`.gcdump` files open directly in Visual Studio (Memory tab) or PerfView. `.dmp` files require `dotnet-dump analyze` (a SOS-like REPL) or WinDbg + SOS.

The right tool for: investigating leaks. The standard workflow is:

1. Take a snapshot after the app warms up (`gcdump 1`)
2. Run the suspected workload (e.g., 1000 requests through the leaky endpoint)
3. Take a second snapshot (`gcdump 2`)
4. Compare — types whose count grew unexpectedly are the candidates

JetBrains dotMemory has the slickest UI for this comparison; PerfView is free and works everywhere.

## PerfView and dotMemory

| Tool | Cost | Strengths |
|---|---|---|
| **PerfView** | Free (Microsoft) | Most complete; handles ETW, GC events, JIT, lock contention. UI is dated. |
| **JetBrains dotMemory** | Commercial | Polished UI, snapshot diff, retention path explorer — best for leak investigation |
| **Visual Studio Diagnostic Tools** | Bundled with VS | Live CPU/memory while debugging; good for first-pass investigation |
| **Visual Studio Performance Profiler** | Bundled with VS | Profile without debugger (more accurate than the debug-time tools) |

For Linux production, `dotnet-trace` + Speedscope covers the same ground as PerfView locally. dotMemory has cross-platform variants too.

## A typical investigation workflow

When something is wrong in production:

1. **Symptom phase** — what does the user see? High latency? OOM? Slow startup?
2. **Live metrics** — `dotnet-counters monitor` to see GC activity, allocation rate, working set, thread pool state
3. **Hypothesis** — based on counters, narrow the suspect: GC-heavy, CPU-heavy, lock-heavy, leaking?
4. **Capture** — `dotnet-trace` for CPU/allocation, `dotnet-gcdump` for memory leaks
5. **Analyze** — PerfView, Speedscope, or dotMemory; identify the top offender
6. **Hypothesis fix** — code change targeting that specific offender
7. **Validate locally** — BenchmarkDotNet comparing before/after on the suspect path
8. **Validate in production** — deploy, watch counters confirm the regression went away

Skipping any of these steps tends to result in optimization that doesn't help — or worse, that hurts.

## What allocations look like in a profile

Common offenders that show up in allocation traces of average .NET code:

- `String` and `Char[]` — concatenation, `Substring`, `ToString` on numbers
- `Task<T>` — async methods on the slow path
- `List<T>` and its underlying `T[]` — capacity churn
- Boxing of value types — `Int32`, `Boolean`, etc. appearing as heap allocations
- Closure / `<>c__DisplayClass` — lambdas capturing locals
- Exception objects — exceptions are slow paths everywhere

If your top allocators are these, the standard remedies (`StringBuilder`, `ValueTask`, `ArrayPool`, generic collections, `static` lambdas, fast-path validation) usually solve 80% of the problem. The remaining 20% is where you reach for `Span<T>`, `Memory<T>`, custom pooling, and unsafe constructs — and only after measuring confirms the gain.

---

[← Previous: JIT and Runtime](07-jit-and-runtime.md) | [Back to index](README.md) | [Next: Pipelines →](09-pipelines.md)
