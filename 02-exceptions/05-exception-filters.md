# Exception Filters (`when`)

Exception filters are a C# 6+ feature that adds a `when (predicate)` clause to a `catch`. They look like sugar but have a real semantic difference from `catch + if + throw`: the **stack is not unwound** while the predicate runs. That's the senior-level detail.

## Syntax

```csharp
try
{
    // ...
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    // handle 404
}
catch (HttpRequestException ex) when (IsTransient(ex))
{
    // retry policy
}
catch (HttpRequestException ex)
{
    // everything else
}
```

The catch matches only when both the type matches **and** the `when` predicate returns `true`. If the predicate returns `false` (or itself throws), the catch is skipped and the runtime continues searching up the stack.

## Why filters beat `catch + if + throw`

Without filters, the equivalent looks like:

```csharp
catch (HttpRequestException ex)
{
    if (ex.StatusCode == HttpStatusCode.NotFound)
    {
        // handle
    }
    else
    {
        throw;   // rethrow if we shouldn't handle this case
    }
}
```

This works, but per the official C# docs:

> "Exception filters (`when`): The filter expression is evaluated *before* the stack is unwound. This means the original call stack and all local variables remain intact during filter evaluation."
>
> "Traditional `catch` blocks: The catch block runs *after* the stack is unwound, potentially losing valuable debugging information."

Two concrete consequences:

### 1. Better debugger experience

When a `when` filter returns `false`, the runtime keeps walking up the stack as if the catch wasn't there. The original throw site, all local variables in the throwing frame, and the full stack are still live. Hit the breakpoint or first-chance exception in your debugger and you see the actual point of failure.

With `catch + if + throw`, the runtime first unwinds into the catch (popping the original frame), then `throw;` starts a fresh propagation. By the time the next `catch` matches, the original locals and intermediate frames are gone.

### 2. No stack-trace pollution

`throw;` doesn't *change* the stack trace, but it adds a "rethrow" notation. With filters, the exception keeps propagating without ever being caught — the stack trace points cleanly to the original throw site.

## Filters vs base-derived ordering

Without filters, more derived exception types had to come first. With filters, you can have multiple `catch` clauses for the **same** exception type, distinguished by `when`:

```csharp
try
{
    DoWork();
}
catch (Exception ex) when (ex is ArgumentException || ex is FormatException)
{
    return Result.BadInput();
}
catch (Exception ex) when (IsTransient(ex))
{
    return Result.Retry();
}
catch (Exception ex)
{
    return Result.Fatal();
}
```

Per the official docs:

> "If a `catch` clause has an exception filter, it can specify the exception type that is the same as or less derived than an exception type of a `catch` clause that appears after it. For example, if an exception filter is present, a `catch (Exception e)` clause doesn't need to be the last clause."

## Using filters for diagnostic side effects (carefully)

A common idiom is logging *before* deciding whether to handle:

```csharp
catch (Exception ex) when (LogAndContinue(ex))
{
    // never runs — LogAndContinue always returns false
}

private bool LogAndContinue(Exception ex)
{
    _logger.LogError(ex, "Unhandled in {Method}", _operationName);
    return false;
}
```

The filter returns `false`, so the catch never matches and the exception keeps propagating. But `LogAndContinue` runs **with the stack still intact**, so the log captures the original throw site, full locals, the works.

Compare with the without-filter version:

```csharp
catch (Exception ex)
{
    _logger.LogError(ex, "Unhandled");   // stack already unwound
    throw;
}
```

Both log the exception. The filter version logs from the original frame; the catch version logs after popping into the catch block. For first-chance exceptions and high-fidelity diagnostics, the filter is better.

> Be careful with the side-effect-in-filter pattern. The Framework Design Guidelines warn: *"DO NOT throw exceptions from exception filter blocks. When an exception filter raises an exception, the exception is caught by the CLR, and the filter returns false."* If your filter has any chance of throwing, swallow it inside the filter or you'll silently get filter-returns-false behavior that's hard to debug.

## Filter is also evaluated for *every* matching catch up the stack

Filters run as the runtime searches for a handler. If a filter throws, the original exception keeps propagating (the filter's exception is silently dropped). If you have side effects in filters, design them to be cheap and idempotent — they may run multiple times in nested try blocks during the search.

## Filtering on cancellation correctly

The standard pattern for distinguishing cancellation from other failures:

```csharp
try
{
    await DoWorkAsync(ct);
}
catch (OperationCanceledException) when (ct.IsCancellationRequested)
{
    // genuine cancellation — exit cleanly
}
catch (OperationCanceledException ex)
{
    // OCE thrown for a reason other than this token — treat as failure
    _logger.LogWarning(ex, "Unexpected OCE not tied to our token");
    throw;
}
```

This separates two cases that look identical without the filter:

1. The user/caller cancelled via the token we own → expected, exit gracefully.
2. Some inner code threw an OCE for an unrelated cancellation → unexpected, log and bubble.

## Filter doesn't change the catch order; it constrains it

The runtime still evaluates catches top-to-bottom. The filter only adds a `&&`. So you can't use a filter to "fall through" — once the type matches and the filter is true, the catch runs and that's it.

## Senior-interview gotchas

- **Filters evaluate *before* stack unwinding.** That's the structural difference vs `catch + if + throw`.
- **Better debugging** — first-chance exceptions and the debugger see the original throw site and locals.
- **Stack-trace cleanliness** — when the filter returns `false`, no rethrow is recorded.
- **Multiple catches for the same type** — only legal when distinguished by filters.
- **A filter that throws silently returns `false`.** Keep filters cheap and exception-free.
- **Use filter for `OperationCanceledException` + `ct.IsCancellationRequested`** to separate genuine cancellation from unrelated OCEs.
- **Don't put real work in a filter** — if the predicate has more than ~5 lines, refactor.

## Useful Links

- [Exception filters — C# reference (MS Learn)](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/exception-handling-statements#a-when-exception-filter) — the `when` clause, advantages, when to use
- [Exception throwing — Framework Design Guidelines](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/exception-throwing) — guidance on filter behavior

---

[← Previous: Custom Exceptions](04-custom-exceptions.md) | [Back to index](README.md) | [Next: ExceptionDispatchInfo →](06-dispatch-info.md)
