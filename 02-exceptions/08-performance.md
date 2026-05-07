# Performance

Throwing an exception is **expensive**. Most application code can ignore this — exceptions are for the rare error path. But a senior should know the cost model so they recognize when "exceptions for control flow" hurts production.

## Why throwing is slow

When you `throw`, the runtime has to:

1. **Build the exception object** — allocate heap memory, run the constructor, set `Message`, `InnerException`, `Data`, etc.
2. **Capture the stack trace** — walk the call stack frame-by-frame, resolve method names, record file/line info if PDBs are loaded. This is the dominant cost on first throw.
3. **Walk the stack searching for a handler** — for each frame, evaluate any exception filters, check catch clause types.
4. **Unwind the stack** — run `finally` blocks and `using` disposal in reverse order until the matching catch is reached.

On modern .NET, a thrown-and-caught exception costs **~tens to hundreds of microseconds** depending on stack depth. That sounds tiny, but in tight loops or hot paths it dominates everything else.

The official Framework Design Guidelines:

> "CONSIDER the performance implications of throwing exceptions. Throw rates above 100 per second are likely to noticeably impact the performance of most applications."

100 throws/second is the rough threshold where "exceptions are free for the happy path" starts breaking down.

## Catching is essentially free

A `try`/`catch` block that **never throws** has near-zero cost in modern JITs. The runtime emits a small entry in the method's exception-handling table; no overhead per call. So:

- Wrapping code in `try`/`catch` doesn't slow it down.
- The cost is paid only when an exception actually flies.

This is why "use exceptions for the exceptional case" works: error-handling structure is free, only the error path is slow.

## When the cost matters

Patterns where exception cost dominates:

### 1. Validating user input in a loop

```csharp
// BAD on hot path with many invalid inputs
foreach (var s in inputs)
{
    try { result.Add(int.Parse(s)); }
    catch (FormatException) { /* skip */ }
}

// GOOD
foreach (var s in inputs)
{
    if (int.TryParse(s, out int n)) result.Add(n);
}
```

If 30% of inputs are invalid, the BAD version pays exception cost 30% of the time. The GOOD version pays nothing.

### 2. Searching collections

```csharp
// BAD — relies on KeyNotFoundException
try { return dict[key]; }
catch (KeyNotFoundException) { return default; }

// GOOD
return dict.TryGetValue(key, out var v) ? v : default;
```

Same principle: when the "not found" case is common, the throw cost matters.

### 3. Type checking via cast

```csharp
// BAD on hot paths
try { var t = (T)obj; }
catch (InvalidCastException) { /* fallback */ }

// GOOD
if (obj is T t) { /* use t */ }
```

`as` and `is` are pattern-matching operators that **return null / false** instead of throwing. They're the right tool for "maybe this is the right type".

### 4. Loop control

```csharp
// PATHOLOGICAL
try
{
    while (true) DoWork();
}
catch (NoMoreWorkException) { /* loop exit */ }
```

Throwing to exit a loop is exception-as-flow-control. The cost compounds with every iteration.

## The Tester-Doer and Try-Parse patterns

Two BCL conventions explicitly designed to avoid the throw cost:

### Tester-Doer

A "tester" property/method that lets the caller check before invoking the "doer":

```csharp
if (queue.Count > 0)         // tester
    var item = queue.Dequeue();   // doer (throws if empty)
```

Example pairs in BCL: `IEnumerator.MoveNext`/`Current`, `Stack.Count`/`Pop`, etc.

### Try-Parse

A `Try*` method that returns `bool` and `out`s the result, avoiding throw on failure:

```csharp
if (int.TryParse(input, out int n)) { /* use n */ }
```

Per the official Framework Design Guidelines:

> "There are cases when the Tester-Doer Pattern can have an unacceptable performance overhead. In such cases, the so-called Try-Parse Pattern should be considered."

The Try-Parse pattern is preferred when:

- The check has a race condition (tester can return true, then state changes before doer runs).
- The tester would be doing nearly the same work as the doer (wasteful).

BCL Try-Parse pairs: `int.TryParse`, `Guid.TryParse`, `DateTime.TryParse`, `Dictionary.TryGetValue`, `ConcurrentQueue.TryDequeue`, `Channel.Reader.TryRead`, `Stream.TryGetBuffer`, etc.

## First-throw cost vs subsequent

The first throw of a given type from a given site can be especially slow because the runtime resolves debug info (PDB lookups for source-line locations). Subsequent throws of the same type/site reuse cached metadata and are faster.

This means microbenchmarks of "throw cost" can be misleading. Real production cost is usually closer to the warmed-up rate, but pathological apps that throw thousands of unique exception types/sites pay the cold cost repeatedly.

## `[StackTraceHidden]` and `[DebuggerStepThrough]`

Used by `ArgumentNullException.ThrowIfNull` and similar .NET 6+ helpers — these attributes hide the throw helper from the stack trace, so the trace points at the *caller*, not at the validator inside the BCL. The performance impact is minor (slightly faster stack walking), but the diagnostics improve significantly.

You can apply `[StackTraceHidden]` to your own throw helpers:

```csharp
[StackTraceHidden]
private static void ThrowOrderNotFound(int orderId) =>
    throw new OrderNotFoundException(orderId);
```

The exception still throws normally; the stack trace just skips this frame, so the caller sees the throw "originating" from their own code.

## Don't throw on configuration paths

A subtle anti-pattern: using exceptions in configuration validation that runs at startup of every test or every CLI invocation. Each throw adds latency to startup. Prefer:

```csharp
// SLOW (each invalid key throws)
try { value = config[key]; }
catch (KeyNotFoundException) { value = defaultValue; }

// FAST
value = config.TryGetValue(key, out var v) ? v : defaultValue;
```

Multiplied across hundreds of config keys × thousands of invocations, the cost is real.

## When the cost doesn't matter

Most application code throws far fewer than 100 exceptions per second. In that regime:

- **Don't optimize prematurely.** Code clarity wins.
- **Don't avoid `try`/`catch`** out of fear — the structure itself is free.
- **Don't compromise the error model** to dodge a cost that's not measurable in your workload.

The cases where throw cost really matters: parsers handling malformed input, dispatchers receiving bad messages, validation loops over user-supplied data. Profile before optimizing.

## Senior-interview gotchas

- **Throwing is expensive (~tens of µs); catching is free.** The cost is paid only when an exception flies.
- **Throw rates above ~100/sec start hurting**, per the Framework Design Guidelines.
- **Use `Try*` patterns on hot paths** — `TryParse`, `TryGetValue`, `TryDequeue`.
- **Use `is` / `as` instead of try-cast.** They don't throw on type mismatch.
- **Don't use exceptions for loop control or normal flow.**
- **First throw of a type/site is slowest** (PDB resolution). Microbenchmarks may understate or overstate real cost.
- **`[StackTraceHidden]`** lets you hide validator frames so the stack trace points at the caller's code.

## Useful Links

- [Exceptions and Performance — Framework Design Guidelines](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/exceptions-and-performance) — Tester-Doer, Try-Parse, throw cost
- [Best practices for exceptions — MS Learn](https://learn.microsoft.com/en-us/dotnet/standard/exceptions/best-practices-for-exceptions) — performance section
- [`StackTraceHiddenAttribute` — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.diagnostics.stacktracehiddenattribute) — for hiding throw helper frames

---

[← Previous: Async Exceptions and AggregateException](07-async-and-aggregate.md) | [Back to index](README.md)
