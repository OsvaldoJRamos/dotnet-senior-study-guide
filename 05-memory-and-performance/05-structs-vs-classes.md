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

## struct vs ref struct vs string

A classic interview trap. `string` *behaves* like a value: it is immutable, and `==` compares content. But it is a **reference type**. `ref struct` is a struct with extra compiler-enforced rules that keep it off the heap.

| | `struct` | `ref struct` | `string` |
|---|---|---|---|
| Kind | Value type | Value type, **stack-only** | Reference type: `public sealed class String` |
| A variable holds | The data itself | The data itself | A **reference** to an object on the heap |
| Assignment / passing an argument | Copies the whole value | Copies the whole value | Copies the reference (cheap) |
| Where the data lives | On the stack, or **inline** in the containing object/array | Stack only. It *"can't escape to the managed heap"* | Managed heap |
| Mutability | Mutable, unless `readonly struct` | Mutable, unless `readonly ref struct`. Careful: `Span<T>` is a `readonly ref struct` whose **elements** are still writable; `ReadOnlySpan<T>` is fully read-only. | **Immutable** |
| Can be `null` | No (only `Nullable<T>` / `T?`) | No | Yes |
| `==` | Not available unless overloaded (`record struct` generates it) | Not available unless overloaded | Compares **values** (case-sensitive ordinal) |
| Boxing to `object` / interface | Yes, allocates on the heap | **Forbidden** | N/A, it's already an object |
| Field of a class / array element | Yes | **No** | Yes |
| Captured by a lambda | Yes | **No** | Yes |
| Used across `await` / `yield return` | Yes | **No** (C# 13+: allowed in async/iterator methods, but not across `await` / `yield return`) | Yes |
| Typical examples | `int`, `DateTime`, `Guid`, small value objects | `Span<T>`, `ReadOnlySpan<T>` | Text |

```csharp
public struct Point { public int X; public int Y; }

var a = new Point { X = 1, Y = 2 };
var b = a;            // copies the value
b.X = 10;             // a.X is still 1

string s1 = "order";
string s2 = s1;       // copies the reference
s2 += "-42";          // creates a NEW string; s1 is still "order"

ReadOnlySpan<char> prefix = "order-42".AsSpan(0, 5); // a view over the string, no allocation
```

Gotchas worth saying out loud in an interview:

- **"string is a value type because `==` compares content."** Wrong. The C# docs: *"Although `string` is a reference type, the equality operators `==` and `!=` compare the values of `string` objects, not references."* The reference type simply overloads equality.
- **Immutability means hidden allocations.** *"Methods that appear to modify a String object actually return a new String object that contains the modification."* In loops, use `StringBuilder`. To slice without copying, use `ReadOnlySpan<char>` via `AsSpan()` (see [Modern String APIs](11-modern-string-apis.md)).
- **Interning is not a guarantee.** The `String.Intern` docs: *"automatic interning of string literals isn't guaranteed—depending on how the assembly was compiled and executed, some literals might not be added to the pool."* Never rely on two equal strings being the same reference.
- **Default struct equality is slow.** `ValueType.Equals` *"calls Object.Equals(Object) on each field"*, and for non-blittable structs *"uses the System.Reflection library to compare the contents of all fields"* (CA1815). Override `Equals`/`GetHashCode` and implement `IEquatable<T>`, or use a `record struct`.
- **Why `ref struct` exists.** A `Span<T>` can point at stack memory (`stackalloc`) or into the middle of an array. If it could be boxed, stored in a field, or kept alive across an `await`, it could outlive the memory it points to. The restrictions make that impossible at compile time. Side effect: on `Span<T>`, `Equals(object)` and `GetHashCode` throw `NotSupportedException`.

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
