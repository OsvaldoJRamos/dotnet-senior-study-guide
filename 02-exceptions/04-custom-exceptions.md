# Custom Exceptions

Most code shouldn't ship custom exception types — the BCL's `ArgumentException`, `InvalidOperationException`, `IOException`, etc. are usually sufficient. But there are cases where a custom type adds real value (programmatic catch boundaries, structured payload). Knowing when and how is part of the senior bar.

## When to create one

Per the official "Best practices" page:

> "Introduce a new exception class only when a predefined one doesn't apply."

Three legitimate reasons to create a custom exception:

1. **Callers need to catch it specifically.** If your code throws and consumers need to differentiate "this kind of failure" from other failures of the same base type, a dedicated subtype is worth it (e.g., `OrderNotFoundException` vs `OrderExpiredException` — both could be `InvalidOperationException`, but callers want different recovery).
2. **Structured payload beyond `Message`.** Diagnostic data that callers will read programmatically (`OrderId`, `RetryCount`, `RemoteEndpoint`). The `Exception.Data` dictionary works for *ad hoc* metadata; typed properties on a custom exception document the contract.
3. **Domain language.** A `ConcurrencyConflictException` in your code reads better than a `DBConcurrencyException` leaking from EF Core's internals — it's a cleaner abstraction boundary.

What's NOT a good reason: "to feel organized", "because it's a different module", or because you want a `MyServiceException` per service. Excessive custom exceptions create noise without benefit.

## Three constructors are the minimum

Per the official guidance:

> "Use at least the three common constructors when creating your own exception classes: the parameterless constructor, a constructor that takes a string message, and a constructor that takes a string message and an inner exception."

```csharp
public class OrderNotFoundException : Exception
{
    public OrderNotFoundException()
    { }

    public OrderNotFoundException(string message)
        : base(message)
    { }

    public OrderNotFoundException(string message, Exception innerException)
        : base(message, innerException)
    { }
}
```

Why all three:

- **Parameterless** — needed by some serializers and frameworks that activate the type via `Activator.CreateInstance`.
- **Message** — the common case.
- **Message + inner** — required for wrapping (preserve the original cause when you catch + rethrow as your domain type).

## Naming

> "End exception class names with `Exception`." — MS Learn

Always. `OrderNotFoundException`, not `OrderNotFound`. The suffix is a convention every .NET developer expects. Exception class names should describe the failure, not the operation that failed.

## Derive directly from `Exception`

```csharp
public class OrderNotFoundException : Exception { ... }
```

NOT `: ApplicationException`. The Microsoft Framework Design Guidelines explicitly recommend deriving from `Exception` directly. `ApplicationException` is a historical artifact — see [Exception Hierarchy](02-exception-hierarchy.md).

If your domain has many related exception types (e.g., a payment service throws ~5 distinct ones), it's reasonable to introduce a domain base class:

```csharp
public abstract class PaymentException : Exception
{
    protected PaymentException() { }
    protected PaymentException(string m) : base(m) { }
    protected PaymentException(string m, Exception inner) : base(m, inner) { }
}

public class CardDeclinedException : PaymentException { ... }
public class PaymentTimeoutException : PaymentException { ... }
```

Callers can `catch (PaymentException)` to handle the whole family without knowing each subtype. This pays off only when there are multiple subtypes; otherwise it's overhead.

## Adding properties

> "Provide more properties for an exception (in addition to the custom message string) only when there's a programmatic scenario where the additional information is useful." — MS Learn

```csharp
public class OrderNotFoundException : Exception
{
    public int OrderId { get; }

    public OrderNotFoundException() { }

    public OrderNotFoundException(int orderId)
        : base($"Order {orderId} was not found.")
    {
        OrderId = orderId;
    }

    public OrderNotFoundException(int orderId, Exception innerException)
        : base($"Order {orderId} was not found.", innerException)
    {
        OrderId = orderId;
    }
}
```

Callers can now write:

```csharp
catch (OrderNotFoundException ex) when (ex.OrderId == requestedId)
{
    // recovery specific to this order
}
```

The example mirrors the BCL: `FileNotFoundException` exposes a `FileName` property, `ArgumentException` exposes `ParamName`, etc.

> If the data is one-off diagnostic context (RequestId, UserId, timestamps), use `Exception.Data` instead of subclassing — `Data` is meant for exactly this. Reserve typed properties for programmatic recovery.

## Serialization (legacy, mostly)

In older code you'll see exceptions implementing the binary serialization contract:

```csharp
[Serializable]
public class MyException : Exception, ISerializable
{
    // ... three standard ctors ...

    protected MyException(SerializationInfo info, StreamingContext context)
        : base(info, context) { }

    public override void GetObjectData(SerializationInfo info, StreamingContext context)
    {
        info.AddValue(nameof(OrderId), OrderId);
        base.GetObjectData(info, context);
    }
}
```

**Status as of .NET 8+:** the binary serialization constructor `Exception(SerializationInfo, StreamingContext)` is **obsolete** (per the `Exception` MS Learn page). Binary serialization is being deprecated across .NET. For new exceptions, drop the `[Serializable]` attribute and the special constructor unless you have a concrete cross-AppDomain or remoting scenario, which is rare in modern code. Exceptions still cross HTTP/gRPC/queue boundaries, but those use JSON/Protobuf serialization of the message payload, not `BinaryFormatter`.

## Don't add behavior

A custom exception is a **data container**, not a service. Don't add methods that "handle" or "log" themselves — that pulls dependencies into the exception type and creates surprising side effects when the exception is constructed in a different context (e.g., a unit test mocks the service, but the exception still tries to log).

```csharp
// BAD
public class OrderException : Exception
{
    public void LogAndRethrow(ILogger logger) { ... }   // belongs elsewhere
}
```

Logging, metrics, and recovery belong to the calling code, not to the exception type.

## Senior-interview gotchas

- **End the class name with `Exception`** — convention, not optional.
- **Three constructors minimum** (parameterless, message, message + inner).
- **Derive from `Exception`, not `ApplicationException`.**
- **Add typed properties only for programmatic recovery.** Use `Exception.Data` for diagnostic metadata.
- **Don't add behavior** — exceptions are data, not services.
- **Binary serialization is obsolete in .NET 8+** — drop the `[Serializable]` attribute and the `(SerializationInfo, StreamingContext)` constructor in new code unless you have a specific need.
- **Domain base class** (e.g., `PaymentException`) is worth it only with 3+ related subtypes that callers actually catch as a group.

## Useful Links

- [Best practices for exceptions — Custom exception types — MS Learn](https://learn.microsoft.com/en-us/dotnet/standard/exceptions/best-practices-for-exceptions#custom-exception-types) — the three-constructor rule, naming, properties
- [How to: Create user-defined exceptions — MS Learn](https://learn.microsoft.com/en-us/dotnet/standard/exceptions/how-to-create-user-defined-exceptions)
- [Design guidelines for exceptions — MS Learn](https://learn.microsoft.com/en-us/dotnet/standard/design-guidelines/exceptions) — Cwalina-Abrams design principles

---

[← Previous: Best Practices](03-best-practices.md) | [Back to index](README.md) | [Next: Exception Filters →](05-exception-filters.md)
