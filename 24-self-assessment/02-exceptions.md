# Exceptions

> Read the questions, think about your answer, then click to reveal.

---

### 1. What's the difference between `throw;` and `throw ex;` inside a `catch` block?

<details>
<summary>Reveal answer</summary>

`throw;` rethrows the current exception **preserving the original stack trace** (including the original throw site). `throw ex;` resets the stack trace to start from the current line — so the original location of failure is lost.

Per the official C# docs: *"`throw;` preserves the original stack trace of the exception, which is stored in the `Exception.StackTrace` property. In contrast, `throw e;` updates the `StackTrace` property of `e`."*

Code analysis rule **CA2200** flags `throw ex;`. The fix is just `throw;`.

If you need to rethrow from outside the `catch` block (different method, different thread), use `ExceptionDispatchInfo.Capture(ex)` + `.Throw()`.

Deep dive: [Fundamentals](../02-exceptions/01-fundamentals.md), [`ExceptionDispatchInfo`](../02-exceptions/06-dispatch-info.md)

</details>

---

### 2. How many `Exception`-derived types should a "catch-all" guard against, and where does it belong?

<details>
<summary>Reveal answer</summary>

**`catch (Exception)` belongs only at outermost boundaries** — HTTP middleware, a `BackgroundService` loop, a message-bus consumer, `Main`. Not sprinkled through application code.

Per MS Learn: *"When your code can't recover from an exception, don't catch that exception. Enable methods further up the call stack to recover if possible."*

The two anti-patterns:

```csharp
// SWALLOW — turns exception into silence
try { _service.Save(o); }
catch (Exception) { }

// LOG-AND-CONTINUE — logged but turns exception into surprise null
try { return _service.Save(o); }
catch (Exception ex) { _logger.LogError(ex, "Failed"); return null; }
```

Both convert exceptions into bugs that surface elsewhere. Catch SPECIFIC types you can recover from; let unknown ones bubble to the boundary.

Reserved exceptions you should NOT catch (or throw): `NullReferenceException`, `IndexOutOfRangeException`, `OutOfMemoryException`, `StackOverflowException` (uncatchable since .NET 2.0). See CA2201.

Deep dive: [Best Practices](../02-exceptions/03-best-practices.md), [Exception Hierarchy](../02-exceptions/02-exception-hierarchy.md)

</details>

---

### 3. What's the difference between an exception filter (`when`) and a `catch + if + throw`?

<details>
<summary>Reveal answer</summary>

A `when` filter is evaluated **before** the stack is unwound. A `catch + if + throw;` evaluates **after** unwinding into the catch block.

Per the official C# docs: *"Exception filters (when): The filter expression is evaluated before the stack is unwound. This means the original call stack and all local variables remain intact during filter evaluation."*

Practical consequences:

1. **Better debugging** — first-chance exceptions and the debugger see the original throw site with all frames and locals intact when the filter runs.
2. **Cleaner stack traces** — when the filter returns false, the exception keeps propagating without recording a "rethrow here" event.

```csharp
// Idiomatic for "cancellation we asked for vs unrelated cancellation":
catch (OperationCanceledException) when (ct.IsCancellationRequested)
{
    // graceful exit
}
catch (OperationCanceledException ex)
{
    // OCE from somewhere else — log and bubble
    _logger.LogWarning(ex, "Unexpected OCE");
    throw;
}
```

Two warnings: a filter that throws silently returns `false` (CLR catches it). And per the Framework Design Guidelines, *"DO NOT throw exceptions from exception filter blocks."*

Deep dive: [Exception Filters](../02-exceptions/05-exception-filters.md)

</details>

---

### 4. When should you create a custom exception type, and what's the minimum boilerplate?

<details>
<summary>Reveal answer</summary>

Per MS Learn: *"Introduce a new exception class only when a predefined one doesn't apply."* Three legitimate reasons:

1. Callers need to catch this kind of failure specifically.
2. Structured payload beyond `Message` (typed properties for programmatic recovery).
3. Domain language — a `ConcurrencyConflictException` reads better than a leaked `DBConcurrencyException`.

Minimum boilerplate (the official three constructors):

```csharp
public class OrderNotFoundException : Exception
{
    public OrderNotFoundException() { }
    public OrderNotFoundException(string message) : base(message) { }
    public OrderNotFoundException(string message, Exception innerException)
        : base(message, innerException) { }
}
```

Rules:

- Name ends with `Exception`.
- Derive from `Exception`, NOT `ApplicationException` (deprecated by convention).
- All three constructors (parameterless for activators, message, message+inner for wrapping).
- Add typed properties (e.g., `OrderId`) only for programmatic recovery; use `Exception.Data` for one-off diagnostic context.
- Don't add behavior — exceptions are data, not services.

Note: as of .NET 8+, the binary-serialization constructor `(SerializationInfo, StreamingContext)` is obsolete. Drop the `[Serializable]` attribute and the special constructor in new code.

Deep dive: [Custom Exceptions](../02-exceptions/04-custom-exceptions.md)

</details>

---

### 5. What does `await` do differently from `.Result` regarding exceptions?

<details>
<summary>Reveal answer</summary>

`await` rethrows the **original** exception, **unwrapped**. The runtime uses `ExceptionDispatchInfo` internally so the type and stack trace match what you'd see if the exception had thrown synchronously, plus a `--- End of stack trace from previous location ---` separator.

`.Result` and `.Wait()` always wrap in `AggregateException`. From the official `Task.Wait` docs: *"`AggregateException`: The task was canceled. The `InnerExceptions` collection contains a `TaskCanceledException` object. -or- An exception was thrown during the execution of the task. The `InnerExceptions` collection contains information about the exception or exceptions."*

So `catch (InvalidOperationException)` works after `await` but NOT after `.Wait()` (which would need `catch (AggregateException)` and a drill into `InnerException`).

The lesser-evil compromise when you must block: `task.GetAwaiter().GetResult()` — same blocking cost, but unwrapped exceptions (it's what `await` uses internally).

Deep dive: [Async Exceptions and AggregateException](../02-exceptions/07-async-and-aggregate.md)

</details>

---

### 6. Why is `AggregateException.Flatten()` useful?

<details>
<summary>Reveal answer</summary>

`AggregateException` can nest — `Parallel.For` of `Parallel.For`, or `Task.WhenAll` of `Task.WhenAll`, can produce an aggregate whose inner exceptions contain other aggregates. Walking `InnerExceptions` manually requires recursion.

`Flatten()` returns a **new** `AggregateException` with all nested aggregates recursively unwrapped into a flat `InnerExceptions` collection:

```csharp
catch (AggregateException ae)
{
    foreach (var inner in ae.Flatten().InnerExceptions)
        _logger.LogError(inner, "Failure");
}
```

Note: `await Task.WhenAll(...)` rethrows only the **first** inner exception, not the full aggregate. To get all failures you must inspect `Task.Exception` on the result of `WhenAll`:

```csharp
var all = Task.WhenAll(tasks);
try { await all; }
catch
{
    foreach (var ex in all.Exception!.Flatten().InnerExceptions) { /* ... */ }
    throw;
}
```

Deep dive: [Async Exceptions and AggregateException](../02-exceptions/07-async-and-aggregate.md)

</details>

---

### 7. Why are exceptions slow on the throw path, and where should you avoid them?

<details>
<summary>Reveal answer</summary>

Throwing requires (1) allocating the exception object, (2) **capturing the stack trace** (the dominant cost — walking and resolving frames), (3) walking up the stack to find a handler, (4) running `finally` blocks during unwind. Total: ~tens to hundreds of microseconds per throw.

Per the Framework Design Guidelines: *"Throw rates above 100 per second are likely to noticeably impact the performance of most applications."*

Catching is essentially **free** when no exception flies — modern JIT emits a small EH table entry, no per-call cost.

Where it matters:

- Hot validation loops over user input (use `int.TryParse`, not `int.Parse` + catch).
- Dictionary lookups where misses are common (use `TryGetValue`, not indexer + catch).
- Type tests in hot paths (use `is` / `as`, not `(T)x` + catch).

The patterns to use instead:

- **Try-Parse pattern**: `bool TryParse(string s, out T result)` — no allocation, no throw.
- **Tester-Doer pattern**: check before calling (`if (queue.Count > 0) queue.Dequeue();`).

Don't use exceptions for normal control flow.

Deep dive: [Performance](../02-exceptions/08-performance.md)

</details>

---

### 8. What's the rule for argument validation in `async` methods?

<details>
<summary>Reveal answer</summary>

Validation should throw **synchronously**, before any `await`. From the MS Learn best-practices guide: *"In task-returning methods, you should validate arguments and throw any corresponding exceptions, such as `ArgumentException` and `ArgumentNullException`, before entering the asynchronous part of the method. Exceptions that are thrown in the asynchronous part of the method are stored in the returned task and don't emerge until, for example, the task is awaited."*

If your method has any synchronous validation followed by async I/O, an early validator-throw before the first `await` runs synchronously, raising the exception at the call site (not in `await`).

For methods where you need argument validation to be visible at call time even for callers who store the task without awaiting it immediately, split into a sync wrapper:

```csharp
public Task<Order> GetAsync(int id)
{
    if (id < 0) throw new ArgumentOutOfRangeException(nameof(id));
    return GetAsyncCore(id);
}
private async Task<Order> GetAsyncCore(int id) { /* await */ }
```

This way invalid `id` throws at the call, not inside an eventual `await`.

In .NET 6+, prefer `ArgumentNullException.ThrowIfNull(arg)` and the `ArgumentOutOfRangeException.ThrowIf*` family — same behavior, less code, code-analyzer-recommended.

Deep dive: [Best Practices](../02-exceptions/03-best-practices.md)

</details>

---

[Back to index](README.md)
