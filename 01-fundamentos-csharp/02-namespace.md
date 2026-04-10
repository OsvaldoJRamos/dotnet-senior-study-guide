# Namespace

## What it is

Namespaces serve two main purposes:

- **Logical division** — organize code by grouping classes, structs, and other related types
- **Avoid name conflicts** — of classes, structs, delegates, etc.

## What the namespace means

When you create classes and other types inside a namespace, the namespace actually becomes part of that class's name.

```csharp
namespace EspacoNomes { public class MinhaClasse { } }
```

The actual full name of this class is `EspacoNomes.MinhaClasse`.

## Using classes without specifying the full name

The full class name can be shortened to just `MinhaClasse` by using `using` directives at the top of the file or at the beginning of the namespace:

```csharp
using EspacoNomes; // allows all classes within the namespace to be
// referred to by just the final class name
```

It is also possible to completely rename a class with a using:

```csharp
using NovoNome = EspacoNomes.MinhaClasse;
```

Now you can reference `EspacoNomes.MinhaClasse` by simply using `NovoNome`.

## Direct reference within the same namespace

When code is inside a namespace, it can directly reference everything that is directly in the same namespace. For example, two classes within the same namespace can refer to each other using just the final class name.

## File-scoped namespaces (C# 10+)

Starting with C# 10, you can declare the namespace without braces, applying it to the entire file:

```csharp
namespace MeuProjeto.Services;

public class MeuServico
{
    // The entire file belongs to the MeuProjeto.Services namespace
}
```

This reduces indentation and is the recommended standard in modern projects.

---

[← Previous: .NET Ecosystem](01-ecossistema-dotnet.md) | [Back to index](README.md) | [Next: CLR and IL →](03-clr-e-il.md)
