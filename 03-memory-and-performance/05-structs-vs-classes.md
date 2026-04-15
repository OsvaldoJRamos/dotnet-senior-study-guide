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

---

[← Previous: Memory Leak](04-memory-leak.md) | [Back to index](README.md)
