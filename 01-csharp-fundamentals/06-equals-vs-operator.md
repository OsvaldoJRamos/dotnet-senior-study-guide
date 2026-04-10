# Equals() vs ==

## Fundamental difference

- `==` compares by **reference** (for reference types) or by **value** (for value types)
- `Equals()` compares by **value/content** (can be overridden)

## With value types (int, struct, etc.)

Both compare by **value** - they work the same way:

```csharp
int a = 5;
int b = 5;
Console.WriteLine(a == b);        // True
Console.WriteLine(a.Equals(b));   // True
```

## With reference types (classes)

`==` compares whether the **references point to the same object in memory**:

```csharp
var obj1 = new Person("John");
var obj2 = new Person("John");

Console.WriteLine(obj1 == obj2);      // False (different references)
Console.WriteLine(obj1.Equals(obj2)); // False (default Equals also compares reference)
```

To compare by content, you need to override `Equals()`:

```csharp
public class Person
{
    public string Name { get; set; }

    public override bool Equals(object? obj)
    {
        if (obj is not Person other) return false;
        return Name == other.Name;
    }

    public override int GetHashCode() => Name.GetHashCode();
}

var obj1 = new Person { Name = "John" };
var obj2 = new Person { Name = "John" };
Console.WriteLine(obj1.Equals(obj2)); // True (now compares by content)
```

## Special case: string

`string` is a reference type, but `==` has been **overridden** to compare by content:

```csharp
string a = "hello";
string b = "hello";
Console.WriteLine(a == b);      // True (compares content)
Console.WriteLine(a.Equals(b)); // True
```

## Records (C# 9+)

Records automatically implement `Equals()` and `==` by value:

```csharp
public record Person(string Name);

var p1 = new Person("John");
var p2 = new Person("John");
Console.WriteLine(p1 == p2);      // True
Console.WriteLine(p1.Equals(p2)); // True
```

## Watch out for null

```csharp
Person? p = null;
// p.Equals(other) -> NullReferenceException!
// p == null        -> True (safe)

// Safe approach:
Console.WriteLine(Equals(p, other)); // static method, safe with null
```

## Summary

| Scenario | `==` | `Equals()` |
|----------|------|------------|
| Value types | Value | Value |
| Reference types (default) | Reference | Reference |
| string | Content (overridden) | Content |
| Record | Content (overridden) | Content |
| With Equals override | Reference (unless you also overload `==`) | Custom content |

---

[← Previous: Modifiers](05-modifiers.md) | [Next: Generics →](07-generics.md) | [Back to index](README.md)
