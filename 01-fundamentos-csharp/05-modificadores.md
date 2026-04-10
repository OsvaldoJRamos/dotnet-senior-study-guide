# Modifiers in C#

## Access Modifiers (Methods and Members)

### 1. `public`
- **Usage:** Unrestricted access. Can be used anywhere.
- **When to avoid:** If you don't want to expose internal application details.

### 2. `private`
- **Usage:** Access limited to the class itself.
- **When to use:** To hide internal details, such as helper variables and implementation methods.

### 3. `protected`
- **Usage:** Access within the class itself and derived classes.
- **When to use:** When subclasses need controlled access to members.

### 4. `internal`
- **Usage:** Access allowed only within the same assembly (project).

### 5. `protected internal`
- **Usage:** Access within the same assembly **or** through inheritance.

### 6. `private protected` (C# 7.2+)
- **Usage:** Access only through inheritance **and** within the same assembly.

### Visual summary

```
Modifier             | Same class | Derived class (same assembly) | Same assembly  | Derived class (other assembly) | Anywhere
---------------------|------------|-------------------------------|----------------|-------------------------------|---------------
public               | ✅          | ✅                             | ✅              | ✅                             | ✅
private              | ✅          | ❌                             | ❌              | ❌                             | ❌
protected            | ✅          | ✅                             | ❌              | ✅                             | ❌
internal             | ✅          | ✅                             | ✅              | ❌                             | ❌
protected internal   | ✅          | ✅                             | ✅              | ✅                             | ❌
private protected    | ✅          | ✅                             | ❌              | ❌                             | ❌
```

## Class Modifiers

### 1. `static`
- **Usage:** Members or classes that belong to the type, not to an instance.

### 2. `abstract`
- **Usage:** Defines a mandatory signature in base classes.
- **When to use:** In classes that are just "templates", with logic left to subclasses.

### 3. `virtual` / `override`
- `virtual` indicates that a method or property can be overridden by an inheriting class.
- `override` is used to override.
- **Usage:** To allow methods to be overridden.

### 4. `sealed`
- **Usage:** Prevents a class or method from being inherited/overridden.

## Field Modifiers

### 1. `readonly`
- **Usage:** Defines that a field can only be assigned in the constructor or at declaration.

### 2. `const`
- **Usage:** Defines a fixed value at compile time.
- **When to use:** For fixed values (e.g., `PI`, timeout).
- **When to avoid:** If the value may depend on the environment or change over time.

### 3. `partial`
- **Usage:** Allows splitting the definition of a class/method/struct across multiple files.

## Interface vs Abstract Class

| Characteristic | Interface | Abstract Class |
|---|---|---|
| Multiple inheritance | Yes (can implement several) | No (can only inherit from one) |
| Constructors | No | Yes |
| Fields (state) | No (only properties via contract) | Yes |
| Default implementation | Yes (C# 8+ default methods) | Yes |
| When to use | Define contracts that multiple unrelated classes implement | When there is shared code between related classes |

**Rule of thumb:** you can only extend one class (abstract or not), but you can implement multiple interfaces.

```csharp
// Interface - contrato
public interface INotificavel
{
    void Notificar(string mensagem);
}

// Classe abstrata - comportamento compartilhado
public abstract class EntidadeBase
{
    public Guid Id { get; } = Guid.NewGuid();
    public DateTime CriadoEm { get; } = DateTime.UtcNow;
    
    public abstract void Validar();
}

// Pode herdar de uma classe E implementar múltiplas interfaces
public class Usuario : EntidadeBase, INotificavel
{
    public string Nome { get; set; }
    
    public override void Validar()
    {
        if (string.IsNullOrEmpty(Nome))
            throw new InvalidOperationException("Nome é obrigatório");
    }
    
    public void Notificar(string mensagem)
    {
        Console.WriteLine($"Notificação para {Nome}: {mensagem}");
    }
}
```

## Equals() vs ==

### Operator `==`
- For **value types** (`int`, `double`, `struct`): compares **values**.
- For **reference types** (`class`): compares **references** (whether they point to the same object in memory).
- `string` is an exception: `==` compares the **content** (because the operator is overloaded).

### Method `Equals()`
- Can be **overridden** to define custom equality.
- By default in classes: compares reference (same as `==`).
- By default in structs: compares values field by field (but with reflection -- slow).

```csharp
var a = new Pessoa("João");
var b = new Pessoa("João");

Console.WriteLine(a == b);      // False (referências diferentes)
Console.WriteLine(a.Equals(b)); // False (por padrão, compara referência)

// Após sobrescrever Equals:
public override bool Equals(object obj)
{
    return obj is Pessoa p && p.Nome == Nome;
}

Console.WriteLine(a.Equals(b)); // True (agora compara por valor)
```

**Best practice:** when overriding `Equals`, always override `GetHashCode` as well.

---

[← Previous: Numeric Types](04-tipos-numericos.md) | [Next: Equals vs == →](06-equals-vs-operador.md) | [Back to index](README.md)
