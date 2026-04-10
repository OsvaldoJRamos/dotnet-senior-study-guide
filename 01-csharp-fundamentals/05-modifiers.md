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
// Interface - contract
public interface INotifiable
{
    void Notify(string message);
}

// Abstract class - shared behavior
public abstract class EntityBase
{
    public Guid Id { get; } = Guid.NewGuid();
    public DateTime CreatedAt { get; } = DateTime.UtcNow;
    
    public abstract void Validate();
}

// Can inherit from a class AND implement multiple interfaces
public class User : EntityBase, INotifiable
{
    public string Name { get; set; }
    
    public override void Validate()
    {
        if (string.IsNullOrEmpty(Name))
            throw new InvalidOperationException("Name is required");
    }
    
    public void Notify(string message)
    {
        Console.WriteLine($"Notification for {Name}: {message}");
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
var a = new Person("John");
var b = new Person("John");

Console.WriteLine(a == b);      // False (different references)
Console.WriteLine(a.Equals(b)); // False (by default, compares reference)

// After overriding Equals:
public override bool Equals(object obj)
{
    return obj is Person p && p.Name == Name;
}

Console.WriteLine(a.Equals(b)); // True (now compares by value)
```

**Best practice:** when overriding `Equals`, always override `GetHashCode` as well.

---

[← Previous: Numeric Types](04-numeric-types.md) | [Next: Equals vs == →](06-equals-vs-operator.md) | [Back to index](README.md)
