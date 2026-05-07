# Modern String APIs

The .NET string story has been quietly rewritten over the last few releases. Five APIs in particular are differentiators in modern .NET that show up in performance reviews and interviews: `SearchValues<T>`, `string.Create<TState>`, `DefaultInterpolatedStringHandler`, custom interpolated string handlers, and UTF-8 string literals (`u8`).

## `SearchValues<T>` — the modern `IndexOfAny`

Introduced in **.NET 8** (per the official `System.Buffers.SearchValues` docs). It replaces `IndexOfAny(char[])` with a pre-computed structure optimized for the specific set you're searching for.

The factory is the static `SearchValues` class with overloads for `char`, `byte`, and `string`:

```csharp
using System.Buffers;

private static readonly SearchValues<char> _vowels =
    SearchValues.Create("aeiouAEIOU");

public bool ContainsVowel(ReadOnlySpan<char> text) =>
    text.IndexOfAny(_vowels) != -1;
```

### Why it's faster

Per the official docs, "instances are optimized for situations where the same set of values is frequently used for searching at run time." On creation, `SearchValues.Create` analyzes the set and picks the best strategy: bitmap, range check, vectorized SIMD scan, or specialized hashing. At call sites, `IndexOfAny` overloads accepting `SearchValues<T>` use that pre-computed structure instead of repeatedly computing it.

### Verified overloads

```csharp
public static SearchValues<byte>   Create(ReadOnlySpan<byte> values);
public static SearchValues<char>   Create(ReadOnlySpan<char> values);
public static SearchValues<string> Create(ReadOnlySpan<string> values, StringComparison comparisonType);
```

The `string` overload was added in .NET 9 — multiple-needle string search with the same pre-compute trick.

### When it pays off

- 3+ values to search (for 1-2, plain `IndexOf` is fine and simpler)
- The needle set is **known at startup or compile time** (cache the `SearchValues` instance in a `static readonly` field)
- Inputs are large enough that SIMD has room to win

If you create a new `SearchValues` per call, you're paying the analysis cost without getting the reuse benefit — that's a regression, not an optimization.

## `string.Create<TState>` — build a string with no intermediate allocation

When you know the final length and the data you need to fill it, you can write straight into the string's underlying buffer instead of going through `StringBuilder` or concatenation.

```csharp
public string FormatOrderId(long sequence) =>
    string.Create(16, sequence, (Span<char> span, long seq) =>
    {
        "ORD-".CopyTo(span);
        seq.TryFormat(span[4..], out _, "D12");
    });
```

The verified signature (per MS Learn, available since .NET Core 2.1):

```csharp
public static string Create<TState>(
    int length,
    TState state,
    SpanAction<char, TState> action);
```

### What the docs explicitly warn about

> "The initial content of the destination span passed to action is undefined. Therefore, it is the delegate's responsibility to ensure that every element of the span is assigned. Otherwise, the resulting string could contain random characters."

If your callback fills only part of the buffer, the rest contains whatever bytes were on the heap at allocation time. That's a security concern (data leak) as well as a correctness one.

The docs also note an interop detail: the runtime guarantees the underlying buffer is at least 1 char larger than the span passed to your callback, and that extra slot is reserved for a null terminator. **Writing anything other than `'\0'` to that extra slot is undefined behavior.**

### When it pays off

- You know the exact final length
- You're in a hot path where `StringBuilder` allocation matters
- The format is small enough that the callback is straightforward to write

For larger string assembly with unknown length, `StringBuilder` (pooled if hot) is still the right tool. `string.Create` is a precision instrument.

## `DefaultInterpolatedStringHandler` — what `$"..."` actually compiles to in C# 10+

Introduced in **.NET 6** (per the official `System.Runtime.CompilerServices.DefaultInterpolatedStringHandler` reference). It is a `ref struct` that the C# compiler uses to lower interpolated strings starting in C# 10.

Before C# 10, `$"User {id} did {action}"` compiled to `string.Format("User {0} did {1}", id, action)` — which boxes value types into an `object[]` and incurs all the formatting overhead even if the string is never used.

In C# 10+, the compiler uses `DefaultInterpolatedStringHandler` instead:

```csharp
// What you write:
string s = $"User {id} did {action}";

// What the compiler emits (conceptually):
var handler = new DefaultInterpolatedStringHandler(literalLength: 12, formattedCount: 2);
handler.AppendLiteral("User ");
handler.AppendFormatted(id);
handler.AppendLiteral(" did ");
handler.AppendFormatted(action);
string s = handler.ToStringAndClear();
```

`AppendFormatted<T>(T)` is generic — no boxing of value types. Internal buffer is rented from `ArrayPool<char>` for larger strings. Performance is dramatically better than the pre-C# 10 path for almost no source change.

You don't write `DefaultInterpolatedStringHandler` directly — that's the compiler's job. But knowing the lowered form explains why interpolated strings stopped being a perf concern in modern .NET.

## Custom interpolated string handlers — pay-only-if-used logging

The same handler mechanism is **extension-friendly**: you can write your own handler type and ask the compiler to use it for specific call sites. The classic application is logging:

```csharp
// Without a custom handler:
logger.LogDebug($"Expensive: {ComputeExpensiveDiagnostic()}");
// ComputeExpensiveDiagnostic() always runs, even if Debug is disabled.

// With a custom handler that respects the log level:
public static class LoggerExtensions
{
    public static void LogDebug(
        this ILogger logger,
        [InterpolatedStringHandlerArgument("logger")] ref LogDebugHandler handler)
    {
        if (handler.IsEnabled)
            logger.Log(LogLevel.Debug, handler.GetFormattedText());
    }
}

[InterpolatedStringHandler]
public ref struct LogDebugHandler
{
    public bool IsEnabled { get; }
    private DefaultInterpolatedStringHandler _inner;

    public LogDebugHandler(int literalLength, int formattedCount, ILogger logger, out bool isEnabled)
    {
        IsEnabled = isEnabled = logger.IsEnabled(LogLevel.Debug);
        _inner = isEnabled ? new(literalLength, formattedCount) : default;
    }

    public void AppendLiteral(string s)             { if (IsEnabled) _inner.AppendLiteral(s); }
    public void AppendFormatted<T>(T value)         { if (IsEnabled) _inner.AppendFormatted(value); }
    public string GetFormattedText()                => _inner.ToStringAndClear();
}
```

When `LogDebug` is called and Debug is disabled, the handler short-circuits in the constructor — `ComputeExpensiveDiagnostic()` is **never called**, the format string is never constructed, no allocation happens.

For real production logging in ASP.NET Core, prefer the `[LoggerMessage]` source generator from `Microsoft.Extensions.Logging`. It generates similar zero-overhead-when-disabled code automatically and integrates with structured logging.

## UTF-8 string literals (`u8`) — C# 11

A small but high-impact feature. The `u8` suffix on a string literal produces a `ReadOnlySpan<byte>` containing the UTF-8 byte representation, computed at compile time:

```csharp
ReadOnlySpan<byte> auth = "AUTH "u8;
ReadOnlySpan<byte> http = "HTTP/1.1 "u8;
```

Per the official C# 11 spec (`u8` literals proposal):

> "When the `u8` suffix is used, the value of the literal is a `ReadOnlySpan<byte>` containing a UTF-8 byte representation of the string."

The compiler stores the bytes in the assembly's `.data` section, so the literal is **allocation-free at the call site** — same cost as referencing a static byte array, with the convenience of writing the source text inline.

### What you used to write before C# 11

```csharp
// Pre-C# 11 — manual byte arrays for hot HTTP paths
private static readonly byte[] s_authBytes = { 0x41, 0x55, 0x54, 0x48, 0x20 };
// or, with allocation cost on every startup:
private static readonly byte[] s_authBytes = Encoding.UTF8.GetBytes("AUTH ");
```

### What you write now

```csharp
private static ReadOnlySpan<byte> AuthBytes => "AUTH "u8;
```

The expression-bodied property pattern is the canonical idiom — `static readonly` fields can't be `ReadOnlySpan<byte>` (it's a `ref struct`), but a property with `=> "literal"u8` works and the JIT folds it to the same constant lookup.

### Where this matters

`Utf8JsonReader.ValueTextEquals(ReadOnlySpan<byte>)`, `Encoding.UTF8.GetString(ReadOnlySpan<byte>)`, and many other hot-path APIs accept UTF-8 directly. Writing `reader.ValueTextEquals("name"u8)` is faster than `reader.ValueTextEquals("name")` because the latter has to convert UTF-16 → UTF-8 on every call.

The suffix is **case-insensitive**: `"hello"u8` and `"hello"U8` are equivalent (per the spec).

## Summary table

| API | Introduced | What it replaces | When to use |
|---|---|---|---|
| `SearchValues<T>` | .NET 8 | `IndexOfAny(char[])` | 3+ needles, set known at startup, large input |
| `string.Create<TState>` | .NET Core 2.1 | `StringBuilder` for known-length strings | Known length, hot path, simple format |
| `DefaultInterpolatedStringHandler` | .NET 6 (C# 10) | `string.Format` lowering | Implicit — just use `$"..."` in C# 10+ |
| Custom interpolated handlers | C# 10 | Manual `if (logger.IsEnabled)` guards | Logging hot paths, expensive-to-format args |
| `u8` string literals | C# 11 | Manual `byte[]` arrays / `Encoding.UTF8.GetBytes` | UTF-8 byte literals in protocol code |

These five APIs are how modern .NET makes "string-heavy" code stop being a profiler hot spot without rewriting it in `Span<byte>` from scratch.

---

[← Previous: Memory Model](10-memory-model.md) | [Back to index](README.md)
