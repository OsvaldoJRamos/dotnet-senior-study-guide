# JIT and Runtime Optimizations

The .NET JIT has gained substantial autonomy over the last few releases. Code you write today benefits from optimizations that didn't exist when many production apps were originally authored — knowing what they are helps you (a) read benchmarks correctly and (b) avoid working against the JIT.

## Tiered compilation

Methods are compiled twice (when warranted):

- **Tier 0 (Quick JIT):** generates code quickly with minimal optimizations. Good for startup.
- **Tier 1:** when the runtime sees a method called frequently, it recompiles it with full optimizations.

Verified history (per the official `compilation` runtime config docs):

- **.NET Core 2.1 / 2.2:** tiered compilation **disabled** by default
- **.NET Core 3.0 and later:** tiered compilation **enabled** by default

You don't have to configure anything — it just works. The setting names if you ever need to disable for investigation:

- `runtimeconfig.json`: `System.Runtime.TieredCompilation`
- MSBuild: `<TieredCompilation>false</TieredCompilation>`
- Env var: `DOTNET_TieredCompilation`

### Quick JIT for loops

A separate knob: by default, **Quick JIT does not apply to methods that contain loops**. The reason: a long-running loop stuck in unoptimized Tier 0 code stays unoptimized — there's no method-call boundary for the runtime to swap in the optimized version. So loops skip Tier 0 entirely and go straight to optimized compilation, trading startup time for steady-state speed.

You can opt in (turn Tier 0 back on for loops) if startup latency matters more:

```xml
<TieredCompilationQuickJitForLoops>true</TieredCompilationQuickJitForLoops>
```

### Implication for benchmarking

Any microbenchmark needs a **warm-up phase** to ensure you're measuring Tier 1 code, not Tier 0. BenchmarkDotNet's default Job already does this — but if you roll your own with `Stopwatch`, your numbers are likely garbage.

## Dynamic PGO (Profile-Guided Optimization)

PGO instruments running code, observes which types and branches show up most, and feeds that information into the Tier-1 recompilation. Verified history (per the official .NET 8 performance announcement):

- **.NET 6:** Dynamic PGO previewed, **off by default**
- **.NET 7:** improved, still **off by default**
- **.NET 8:** *"it's now on by default"* — quoting the official announcement

What this enables:

- **Type specialization** — call sites where one concrete type dominates get specialized for that type
- **Devirtualization** — virtual / interface calls become direct calls when the type is predictable
- **Better inlining** — hot paths inline more aggressively
- **Block reordering** — hot blocks placed for cache locality

If you ever need to disable for diagnosis:

```xml
<TieredPGO>false</TieredPGO>
```

Or env var `DOTNET_TieredPGO=0`.

## Devirtualization

A virtual / interface call traditionally requires a vtable lookup, which also blocks inlining. The JIT can replace the indirect call with a direct call when it can prove the concrete type, or — under PGO — when one type dominates.

Two paths:

1. **Sealed types and final overrides.** If the runtime can see that a class is `sealed` (no further subtype possible) or a method is `sealed override`, devirtualization is automatic.
   ```csharp
   public sealed class Dog : Animal
   {
       public override string Name => "Dog";
   }
   // Dog d = ...; d.Name;  // direct call — JIT knows Dog is sealed
   ```
2. **Guarded devirtualization (with PGO).** When PGO observes that a call site is dominated by one concrete type, the JIT emits a fast-path that checks for that type and calls directly, plus a slow path that falls back to virtual dispatch:
   ```csharp
   foreach (Animal a in animals)        // animals is IEnumerable<Animal>
       Console.WriteLine(a.Name);
   ```
   If 95% of the time `animals` is `List<Dog>`, the JIT can specialize accordingly.

Practical takeaway: prefer `sealed` on classes that aren't designed for inheritance — it's free performance.

## Escape analysis

Escape analysis decides whether an object **escapes** the method that allocated it. If the JIT can prove an object never escapes (not returned, not stored in a field, not passed to a method whose body it can't see), it can stack-allocate it instead of going to the heap, eliminate write barriers, or even decompose it into registers.

Recent state (per Microsoft's runtime announcements):

- **.NET 9:** escape analysis enters with limited stack allocation for narrow patterns
- **.NET 10:** stack allocation widens significantly — small non-escaping arrays, non-escaping delegates with captured state, and field-sensitive analysis combined with PGO can stack-allocate enumerators in `foreach` over `IEnumerable<T>` when one concrete type dominates

The takeaway is the same as for tiered compilation: **write idiomatic code and let the JIT do its job**. Patterns that used to pay an "abstraction tax" in .NET Framework are no longer the same trade-off in modern .NET.

## `[MethodImpl]` attributes — when to reach

The JIT has good defaults. The hints below are real but should be reserved for code where a profile shows the JIT made the wrong call.

```csharp
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public int Add(int a, int b) => a + b;
```

Tells the JIT: "this is small and worth inlining everywhere, ignore your usual heuristics." Useful for tiny wrappers, accessors, helpers in hot loops.

```csharp
[MethodImpl(MethodImplOptions.NoInlining)]
public void DebugHelper() { ... }
```

Forces the JIT to keep this method as a separate frame — easier to find in profilers, useful for diagnostic helpers.

```csharp
[MethodImpl(MethodImplOptions.AggressiveOptimization)]
public int HotMethod(...) { ... }
```

Skips Tier 0 and goes straight to optimized compilation. Useful if you know a method is on the hottest path and you want to pay for full optimization at startup rather than the first few hundred calls running unoptimized. Rare in practice.

> Don't sprinkle `AggressiveInlining` everywhere. Inlining a large method increases code size, hurts I-cache, and can confuse the JIT's other optimizations. Profile first.

## What this means in practice

1. **Tiered compilation is default.** Microbenchmarks need warm-up.
2. **Dynamic PGO is default in .NET 8+.** Hot-path performance keeps improving without code changes.
3. **`sealed` is free.** Stamp it on every class that isn't designed to be inherited from.
4. **Escape analysis keeps growing.** Idiomatic code (LINQ, `foreach`, lambdas) pays less abstraction tax in modern .NET than in .NET Framework — re-benchmark before assuming an old "perf rule" still applies.

---

[← Previous: Async and Memory](06-async-and-memory.md) | [Back to index](README.md) | [Next: Diagnostics →](08-diagnostics.md)
