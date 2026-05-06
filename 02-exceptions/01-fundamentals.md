# Fundamentals

Exceptions are .NET's primary error-reporting mechanism. The official `System.Exception` description: *"Represents errors that occur during application execution."* Knowing the language semantics in detail (especially `throw;` vs `throw e;`) is the first thing senior interviews probe.

## The three-part statement

```csharp
try
{
    // code that may throw
}
catch (SpecificException ex) when (Predicate(ex))   // optional filter
{
    // recover, or rethrow
    throw;
}
catch (Exception ex)
{
    // catch-all (use sparingly)
}
finally
{
    // cleanup that always runs
}
```

A `try` block must be followed by **at least one** `catch` or `finally`. The shapes are:

- `try`/`catch` — handle exceptions
- `try`/`finally` — cleanup only, exception still propagates
- `try`/`catch`/`finally` — both

When an exception is thrown, the runtime walks up the stack looking for the first matching `catch`. If none match, the thread terminates (or the process terminates if the unhandled exception reaches the top).

## `catch` clause matching

Catches are evaluated **top to bottom**. The first matching clause wins. Per the official C# docs:

> "When an exception occurs, the runtime checks catch clauses in the specified order, from top to bottom. At most, only one `catch` block runs for any thrown exception."

Order matters: more derived exception types must come before less derived ones, otherwise the broader catch swallows everything before the specific catch is checked. The compiler will warn (CS1058 / CS0160) when an unreachable catch is detected.

```csharp
try { Process(); }
catch (FileNotFoundException ex) { /* runs for missing files */ }
catch (IOException ex)            { /* runs for other I/O errors */ }
catch (Exception ex)              { /* catch-all; usually a code smell */ }
```

A `catch` clause without an exception type matches **any** exception (`catch { ... }`); equivalent to `catch (Exception)` but without binding the variable. Allowed only as the last clause.

## `throw` semantics

### `throw new X(...)` — start a new exception

```csharp
throw new ArgumentException("amount must be positive", nameof(amount));
```

The stack trace begins at this line.

### `throw;` — rethrow the current exception (only inside a `catch` block)

```csharp
catch (Exception ex)
{
    _logger.LogError(ex, "Failed");
    throw;
}
```

Per the C# docs: *"`throw;` preserves the original stack trace of the exception, which is stored in the `Exception.StackTrace` property."*

### `throw e;` — rethrow with a NEW stack trace ⚠️

```csharp
catch (Exception ex)
{
    _logger.LogError(ex, "Failed");
    throw ex;       // BUG: stack trace now starts at this line
}
```

Per the same docs: *"In contrast, `throw e;` updates the `StackTrace` property of `e`."* The original location of failure is lost. Code analysis rule **CA2200** flags this.

> **Rule:** inside a `catch` block, use `throw;` (no expression). Use `throw new X(...)` only when **wrapping** with a new exception type and including the original via `innerException`.

### `throw` as an expression

```csharp
public string Name
{
    get => _name;
    set => _name = value ?? throw new ArgumentNullException(nameof(value));
}

string first = args.Length >= 1 ? args[0] : throw new ArgumentException("Need an arg.");
```

Available in conditional, null-coalescing, expression-bodied members, and lambdas.

## Wrapping with `innerException`

When you catch an exception and want to surface it with more context, wrap it via the `Exception(string, Exception)` constructor:

```csharp
catch (SqlException sqlEx)
{
    throw new OrderProcessingException("Could not save order", sqlEx);
}
```

The original exception is accessible via `Exception.InnerException`, and its stack trace is preserved. Callers can drill into the chain via `GetBaseException()` (which walks `InnerException` recursively to the root).

## Key properties of `Exception`

From the official `System.Exception` docs:

| Property | Purpose |
|---|---|
| `Message` | "Gets a message that describes the current exception." |
| `StackTrace` | "Gets a string representation of the immediate frames on the call stack." |
| `InnerException` | "Gets the `Exception` instance that caused the current exception." |
| `Data` | "Gets a collection of key/value pairs that provide additional user-defined information about the exception." — useful for attaching context (RequestId, UserId, etc.) without subclassing |
| `HResult` | Numeric code for COM/OS interop scenarios |
| `Source` | Name of the application or object that caused the error |
| `TargetSite` | The method that threw the exception |
| `HelpLink` | URL/path to documentation |

`Data` is underused — it's a `IDictionary` you can decorate with diagnostic context:

```csharp
catch (SqlException ex)
{
    ex.Data["OrderId"] = orderId;
    ex.Data["RetryCount"] = retries;
    throw;
}
```

The data flows with the exception through the stack and shows up in structured logs that serialize `Exception.Data`.

## `finally` semantics

`finally` runs regardless of how control leaves the `try` — normal completion, jump statement (`return`/`break`/`continue`/`goto`), or exception propagation. The C# docs:

> "Whether the `finally` block executes depends on whether the operating system chooses to trigger an exception unwind operation. The only cases where `finally` blocks don't execute involve immediate termination of a program. For example, such a termination might happen because of the `Environment.FailFast` call or an `OverflowException` or `InvalidProgramException` exception."

In practice, `finally` is for resource cleanup. Prefer `using` (or `using` declarations) when the resource implements `IDisposable`/`IAsyncDisposable` — the compiler emits the correct `try`/`finally` pattern automatically.

```csharp
// Manual try/finally
var conn = new SqlConnection(cs);
try { conn.Open(); /* ... */ }
finally { conn.Dispose(); }

// Idiomatic: using
using var conn = new SqlConnection(cs);
conn.Open();
// disposed automatically when scope ends
```

`using` over manual `try`/`finally` is one of the project's hard rules; the compiler-generated form is bug-free.

> Don't throw exceptions from `finally` clauses. The Microsoft Framework Design Guidelines: *"DO NOT throw exceptions from exception filter blocks."* and *"AVOID explicitly throwing exceptions from finally blocks."* Code analysis rule **CA2219** flags this. A throw inside `finally` masks the original exception that triggered the unwind.

## Useful Links

- [`System.Exception` class — MS Learn](https://learn.microsoft.com/en-us/dotnet/api/system.exception) — properties, methods, derived types
- [Exception-handling statements (C# reference) — MS Learn](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/exception-handling-statements) — `try`, `catch`, `finally`, `throw`, exception filters
- [Best practices for exceptions — MS Learn](https://learn.microsoft.com/en-us/dotnet/standard/exceptions/best-practices-for-exceptions) — handling, throwing, custom types

---

[Back to index](README.md) | [Next: Exception Hierarchy →](02-exception-hierarchy.md)
