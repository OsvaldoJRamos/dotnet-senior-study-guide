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
var obj1 = new Pessoa("Joao");
var obj2 = new Pessoa("Joao");

Console.WriteLine(obj1 == obj2);      // False (referencias diferentes)
Console.WriteLine(obj1.Equals(obj2)); // False (Equals padrao tambem compara referencia)
```

To compare by content, you need to override `Equals()`:

```csharp
public class Pessoa
{
    public string Nome { get; set; }

    public override bool Equals(object? obj)
    {
        if (obj is not Pessoa outra) return false;
        return Nome == outra.Nome;
    }

    public override int GetHashCode() => Nome.GetHashCode();
}

var obj1 = new Pessoa { Nome = "Joao" };
var obj2 = new Pessoa { Nome = "Joao" };
Console.WriteLine(obj1.Equals(obj2)); // True (agora compara por conteudo)
```

## Special case: string

`string` is a reference type, but `==` has been **overridden** to compare by content:

```csharp
string a = "hello";
string b = "hello";
Console.WriteLine(a == b);      // True (compara conteudo)
Console.WriteLine(a.Equals(b)); // True
```

## Records (C# 9+)

Records automatically implement `Equals()` and `==` by value:

```csharp
public record Pessoa(string Nome);

var p1 = new Pessoa("Joao");
var p2 = new Pessoa("Joao");
Console.WriteLine(p1 == p2);      // True
Console.WriteLine(p1.Equals(p2)); // True
```

## Watch out for null

```csharp
Pessoa? p = null;
// p.Equals(outro) -> NullReferenceException!
// p == null        -> True (seguro)

// Forma segura:
Console.WriteLine(Equals(p, outro)); // metodo estatico, seguro com null
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
