# Structs vs Classes

## Fundamental differences

| Feature | `struct` (Value Type) | `class` (Reference Type) |
|---|---|---|
| Where it is stored | Inlined in its container — stack when local; heap when a field of a reference type, boxed, or captured by a closure / async state machine | Heap |
| Parameter passing | Copy of the value | Copy of the reference |
| Inheritance | Not supported | Supported |
| Can be null | No (unless `Nullable<T>`) | Yes |
| Garbage Collection | Not needed | Needed |
| Default constructor | Pre-C# 10: implicit parameterless constructor only. C# 10+: you can define your own parameterless constructor and field initializers | Can be customized |
| Performance | Better for small and frequent objects | Better for complex objects |

## When to use struct

Structs are better in scenarios with **many small, short-lived objects**, where avoiding heap allocations makes a significant difference:

- Water simulation with a large array of velocity vectors
- City-building games with many objects of the same behavior (like cars)
- Real-time particle systems
- CPU rendering using a large array of pixels

## When to use class

- When the object needs inheritance
- When the object is large (more than ~16 bytes)
- When identity is needed (two instances with the same values are different things)
- When it needs to be null (`null`)
- In the vast majority of everyday situations

## Practical example

```csharp
// Struct - good for coordinates (small, no identity)
// Marked `readonly` so the compiler enforces immutability.
public readonly struct Point
{
    public double X { get; }
    public double Y { get; }
    
    public Point(double x, double y) => (X, Y) = (x, y);
}
// Note: mutable structs are a common footgun — defensive copies made through
// `readonly` fields, properties, or `in` parameters will silently discard any
// mutation you perform on them.

// Class - good for business entities (identity, inheritance)
public class Customer
{
    public Guid Id { get; }
    public string Name { get; set; }
    public string Email { get; set; }
    
    public Customer(string name, string email)
    {
        Id = Guid.NewGuid();
        Name = name;
        Email = email;
    }
}
```

## Records (C# 9+)

For scenarios where you want **immutability** and **value equality** without using struct:

```csharp
// record class - allocated on the heap but with value equality
public record Address(string Street, string City, string State);

// record struct (C# 10) - allocated on the stack with value equality
public readonly record struct Coordinate(double Lat, double Lon);
```

## `readonly struct` and method-level `readonly`

A `readonly struct` declares that **no member can mutate the instance**. The compiler enforces this for fields (must be `readonly` or compiler-synthesized init-only properties) and for instance methods (cannot modify state).

```csharp
public readonly struct Money
{
    public decimal Amount { get; }
    public string Currency { get; }

    public Money(decimal amount, string currency) => (Amount, Currency) = (amount, currency);

    public Money Add(Money other) =>
        new(Amount + other.Amount, Currency); // returns a new Money, doesn't mutate
}
```

### Why it matters

When you pass a regular (mutable) struct via `in` or `ref readonly`, the compiler may insert **defensive copies** before invoking instance methods — because it cannot prove the method won't mutate the struct. With `readonly struct`, the compiler knows mutation is impossible and skips those copies. The combination `readonly struct` + `in` parameters is the idiomatic zero-cost pass-by-reference pattern for value types.

```csharp
// Zero defensive copies — recommended pattern
public void Process(in Money m) { ... }
```

If you have a mutable struct and only some methods don't mutate, mark those methods `readonly`:

```csharp
public struct Counter
{
    public int Count;
    public readonly int Snapshot() => Count; // no defensive copy needed
    public void Increment() => Count++;
}
```

## `in`, `ref`, and `ref readonly` parameters

| Modifier | Pass by reference? | Caller can read? | Caller must initialize? | Callee can write? |
|---|---|---|---|---|
| `in` | Yes | Yes | Yes | No |
| `ref` | Yes | Yes | Yes | Yes |
| `out` | Yes | After call | No | Yes (must) |
| `ref readonly` (return) | Yes | Yes | — | No (read-only alias) |

`in` is the right choice when you want to pass a **large struct** by reference but disallow mutation. The `in` keyword at the call site is optional but documents intent:

```csharp
LargeStruct big = ...;
Process(in big);   // passes a reference, no copy of bytes
```

`ref readonly` is most useful as a return type — it gives callers an alias to internal data without permitting mutation:

```csharp
private readonly LargeStruct[] _items;
public ref readonly LargeStruct Get(int i) => ref _items[i];

ref readonly LargeStruct slot = ref Get(5);
// slot.Field = 10;  // compile error — ref readonly is read-only
```

## `ref struct`

A `ref struct` is a struct that **cannot escape the stack**. `Span<T>` is the canonical example. The runtime enforces:

- Cannot be a field of a regular class (only of another `ref struct`)
- Cannot cross `await` or `yield`
- Cannot be captured in a lambda
- Cannot be boxed (cannot become `object`)
- Cannot be an array element
- Pre-C# 13: could not be a generic type argument in most cases

C# 13 relaxed several of these restrictions:

- `ref struct` types can now implement interfaces (callable only through generic constraints — boxing is still forbidden)
- The new anti-constraint `where T : allows ref struct` lets generic methods accept `ref struct` types

These features unlock APIs that previously needed parallel `Span<T>` and array overloads.

## Choosing struct vs class — the heuristic

Microsoft's guidance: choose `struct` only when **all** the following hold:

1. The type logically represents a **single value** (no identity)
2. The instance size is small (≲ 16 bytes is the classic guideline; larger structs work but pay for it in copies)
3. It is immutable (`readonly struct`)
4. It will not be boxed frequently
5. It will not be passed through `interface` parameters frequently

Otherwise: `class`. Mutable structs are a notorious source of bugs (silent copy-on-property-access), large structs hurt every method call, and frequently boxed structs lose all the supposed performance benefit.

---

[← Previous: Memory Leak](04-memory-leak.md) | [Back to index](README.md) | [Next: Async and Memory →](06-async-and-memory.md)
