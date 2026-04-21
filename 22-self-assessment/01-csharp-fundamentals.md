# C# Fundamentals

> Read the questions, think about your answer, then click to reveal.

---

### 1. What is the difference between .NET, C#, and the CLR?

<details>
<summary>Reveal answer</summary>

- **C#** is a programming language — syntax, type system, features like `async`, pattern matching, records.
- **.NET** is the platform — the SDK, the runtime, the BCL (`System.*`), the tooling (`dotnet` CLI, MSBuild, NuGet), and higher-level stacks (ASP.NET Core, EF Core).
- **CLR (Common Language Runtime)** is the execution engine inside the runtime — it loads assemblies, JIT-compiles IL to machine code, manages memory (GC), and enforces type safety.

One way to line it up: you **write** C# (language) → it compiles to **IL** in assemblies (platform artifact) → the **CLR** runs those assemblies. Other languages (F#, VB.NET) also target the CLR.

"The .NET ecosystem" also includes adjacent pieces — Roslyn (the compiler platform), the runtime libraries, ASP.NET Core, EF Core, ML.NET. Knowing which belongs where is a senior-level cue.

Deep dive: [.NET Ecosystem](../01-csharp-fundamentals/01-dotnet-ecosystem.md) and [CLR and IL](../01-csharp-fundamentals/03-clr-and-il.md)

</details>

---

### 2. What is IL, and what does JIT compilation do?

<details>
<summary>Reveal answer</summary>

**IL (Intermediate Language)** — also called MSIL or CIL — is the CPU-independent bytecode that C#/F#/VB.NET all compile to. It's stored in the assembly (`.dll`/`.exe`) alongside metadata.

**JIT (Just-In-Time) compilation** — at runtime, the CLR takes the IL for a method and compiles it **to native machine code the first time the method is called**. Subsequent calls run the native code directly.

Consequences:
- The same `.dll` runs on x64, ARM64, Linux, Windows — the CLR produces the right machine code for the host.
- First-call overhead is real. **Tiered compilation** (on by default since .NET Core 3.x) mitigates it by generating a fast, unoptimized version first, then recompiling hot methods with full optimizations.
- **ReadyToRun (R2R)** and **Native AOT** let you pre-compile IL to native ahead of time for faster startup — useful for serverless / CLI tools at the cost of some runtime optimizations.

Deep dive: [CLR and IL](../01-csharp-fundamentals/03-clr-and-il.md)

</details>

---

### 3. When do you use `decimal` vs `double` vs `float` in C#?

<details>
<summary>Reveal answer</summary>

They're not interchangeable — pick based on **precision** vs **range/speed**.

| Type | Size | Kind | Precision | Use for |
|------|------|------|-----------|---------|
| `float` | 32-bit | Binary IEEE 754 | ~7 digits | GPU / graphics / ML where 32-bit floats are the norm |
| `double` | 64-bit | Binary IEEE 754 | ~15–17 digits | Scientific / engineering / statistics |
| `decimal` | 128-bit | Base-10 | 28–29 digits | **Money** and anything where rounding to cents must be exact |

Classic trap: `0.1 + 0.2` in `double` is `0.30000000000000004` because 0.1 has no exact binary representation. In `decimal` it's exactly `0.3` because decimal is base-10.

Always use `decimal` (or a dedicated money type) for financial calculations. Use `double`/`float` for measurements and math; never for currency.

Deep dive: [Numeric Types](../01-csharp-fundamentals/04-numeric-types.md)

</details>

---

### 4. What are the access modifiers in C#, and what does each do?

<details>
<summary>Reveal answer</summary>

Six access modifiers (the last two are combinations):

| Modifier | Visible from |
|----------|--------------|
| `public` | Anywhere — any assembly, any type |
| `private` | Only the containing type (default for class members) |
| `protected` | Containing type + derived types (any assembly) |
| `internal` | Anywhere in the **same assembly** (default for top-level types) |
| `protected internal` | Same assembly **OR** any derived type |
| `private protected` | Same assembly **AND** a derived type |

Some extras worth knowing:
- **File-scoped types** (C# 11, `file class Foo`) are visible only within the same source file — useful in source generators.
- `internal` members can be exposed to specific assemblies via `[InternalsVisibleTo("Other.Tests")]` in the project file — the usual pattern for unit-testing internals.

Rule of thumb: start as restrictive as possible (`private`), widen only when a real consumer needs the access. It's far easier to widen access than to narrow it later.

Deep dive: [Modifiers](../01-csharp-fundamentals/05-modifiers.md)

</details>

---

### 5. What do `sealed`, `abstract`, `virtual`, `override`, and `new` do on class members?

<details>
<summary>Reveal answer</summary>

They control inheritance and polymorphism:

- **`virtual`** — marks a method/property as **overridable** in a derived class. Base class provides a default implementation.
- **`override`** — in a derived class, **replaces** the base implementation. Resolves via the runtime type, not the declared type.
- **`abstract`** — on a class: cannot be instantiated. On a member: has no body; derived classes **must** override.
- **`sealed`** — on a class: cannot be inherited. On an overridden method: prevents **further** overrides down the chain.
- **`new`** (method modifier, not the operator) — **hides** an inherited member rather than overriding it. Resolution uses the declared type. Usually a smell — you probably meant `override`.

```csharp
class Animal {
    public virtual string Speak() => "...";
}
class Dog : Animal {
    public override string Speak() => "Woof";      // polymorphic
    public new string ToString() => "dog";         // hides, compiler warns if implicit
}
sealed class Puppy : Dog { }                       // no further inheritance
```

Deep dive: [Modifiers](../01-csharp-fundamentals/05-modifiers.md)

</details>

---

### 6. What is the difference between `Equals()` and `==` in C#?

<details>
<summary>Reveal answer</summary>

| Scenario | `==` | `Equals()` |
|----------|------|------------|
| Value types | Compares **value** | Compares **value** |
| Reference types (default) | Compares **reference** | Compares **reference** |
| `string` | Compares **content** (overloaded) | Compares **content** (overridden) |
| `record` | Compares **content** (overloaded) | Compares **content** (overridden) |
| Custom `Equals` override | Still compares reference (unless `==` is also overloaded) | Uses your custom logic |

When you override `Equals()`, always also override `GetHashCode()` — otherwise dictionaries and hash sets break.

Deep dive: [Equals vs ==](../01-csharp-fundamentals/06-equals-vs-operator.md)

</details>

---

### 7. Explain generic constraints. What is covariance and contravariance?

<details>
<summary>Reveal answer</summary>

**Constraints** restrict what types can be used as a generic argument:

```csharp
void Process<T>(T item) where T : class, IComparable, new()
```

Common constraints: `where T : struct`, `where T : class`, `where T : new()`, `where T : BaseClass`, `where T : IInterface`.

**Covariance** (`out T`): allows a more derived type to be used. `IEnumerable<Dog>` can be assigned to `IEnumerable<Animal>`.

**Contravariance** (`in T`): allows a less derived type to be used. `Action<Animal>` can be assigned to `Action<Dog>`.

Rule of thumb: `out` = output positions (return types), `in` = input positions (parameters).

Deep dive: [Generics](../01-csharp-fundamentals/07-generics.md)

</details>

---

### 8. What are delegates, Func, Action, and events? How do they relate?

<details>
<summary>Reveal answer</summary>

- **Delegate**: a type-safe function pointer — defines a method signature.
- **`Func<T, TResult>`**: built-in delegate that returns a value. `Func<int, string>` takes an `int`, returns a `string`.
- **`Action<T>`**: built-in delegate that returns `void`. `Action<string>` takes a `string`, returns nothing.
- **Event**: a delegate wrapped with `event` keyword — restricts external code to only `+=` and `-=` (no direct invocation or assignment).

```csharp
Func<int, int, int> add = (a, b) => a + b;
Action<string> log = msg => Console.WriteLine(msg);

// Event — only the declaring class can invoke it
public event EventHandler<OrderEventArgs> OrderPlaced;
```

Deep dive: [Delegates and Events](../01-csharp-fundamentals/08-delegates-and-events.md)

</details>

---

### 9. What is the difference between a record, a class, and a struct?

<details>
<summary>Reveal answer</summary>

| Aspect | `class` | `struct` | `record` | `record struct` |
|--------|---------|----------|----------|-----------------|
| Type | Reference | Value | Reference | Value |
| Equality | Reference | Value | Value (content) | Value (content) |
| Mutable by default | Yes | Yes | No (`init`) | No (`init`) |
| Inheritance | Yes | No | Yes | No |
| Best for | Complex entities | Small, immutable data | DTOs, events, immutable models | Small immutable data with value semantics |

Records give you `Equals`, `GetHashCode`, `ToString`, and `with` expressions for free. Use `record` for data that is defined by its values rather than its identity.

</details>

---

### 10. Why are strings immutable in C#? When should you use StringBuilder?

<details>
<summary>Reveal answer</summary>

Strings are **immutable** — every modification creates a new `string` object. This enables:
- **Thread safety** — no locks needed for shared strings
- **String interning** — the runtime can reuse identical literals
- **Security** — a string passed to a method cannot be tampered with

Use **`StringBuilder`** when concatenating in a loop or building strings dynamically:

```csharp
// Bad — O(n^2) allocations
string result = "";
for (int i = 0; i < 1000; i++)
    result += i.ToString();

// Good — O(n) with StringBuilder
var sb = new StringBuilder();
for (int i = 0; i < 1000; i++)
    sb.Append(i);
```

For a small, known number of concatenations, `string.Concat` or interpolation is fine.

</details>

---

### 11. What are nullable reference types and why were they introduced?

<details>
<summary>Reveal answer</summary>

Enabled with `<Nullable>enable</Nullable>`, nullable reference types add **compile-time null safety**. The compiler warns you when:
- You dereference a possibly-null reference without checking
- You assign `null` to a non-nullable reference type

```csharp
string name = null;    // Warning: assigning null to non-nullable
string? nickname = null; // OK — explicitly nullable

if (nickname is not null)
    Console.WriteLine(nickname.Length); // No warning — null check done
```

They were introduced because `NullReferenceException` is the most common runtime exception. It does **not** change runtime behavior — it is purely a compiler analysis.

</details>

---

### 12. How does pattern matching work in C#? Give practical examples.

<details>
<summary>Reveal answer</summary>

Pattern matching lets you test a value against a shape and extract data. Key patterns:

```csharp
// Type pattern
if (obj is string s) Console.WriteLine(s.Length);

// Switch expression with property pattern
var discount = customer switch
{
    { IsPremium: true, Years: > 5 } => 0.20m,
    { IsPremium: true }             => 0.10m,
    _                               => 0.0m,
};

// Relational + logical patterns
string Classify(int temp) => temp switch
{
    < 0          => "Freezing",
    >= 0 and < 20 => "Cold",
    >= 20 and < 35 => "Warm",
    _             => "Hot",
};
```

Pattern matching replaces long `if-else` chains and makes code more declarative and readable.

</details>

---

### 13. What are `ref`, `out`, and `in` parameters?

<details>
<summary>Reveal answer</summary>

| Keyword | Direction | Must initialize before call | Must assign inside method |
|---------|-----------|---------------------------|--------------------------|
| `ref` | In + Out | Yes | No |
| `out` | Out only | No | Yes |
| `in` | In only (readonly ref) | Yes | No (cannot modify) |

```csharp
void Increment(ref int x) => x++;
void TryParse(string s, out int result) => result = int.Parse(s);
void Display(in DateTime date) => Console.WriteLine(date); // cannot modify date
```

- `ref` is for when you need to read and modify the caller's variable.
- `out` is for methods that return multiple values (like `TryParse`).
- `in` passes large structs by reference without copying, while preventing modification.

</details>

---

### 14. What is the difference between an interface and an abstract class?

<details>
<summary>Reveal answer</summary>

| Aspect | Interface | Abstract Class |
|--------|-----------|----------------|
| Inheritance | Multiple | Single |
| Constructors | No | Yes |
| Fields | No | Yes |
| Default implementation | Yes (C# 8+) | Yes |
| Access modifiers on members | Public by default | Any |
| When to use | Define a contract / capability | Share implementation across a hierarchy |

Use **interfaces** when unrelated types share a behavior (`ISerializable`, `IComparable`). Use **abstract classes** when types share an identity and common implementation (`Animal` -> `Dog`, `Cat`).

</details>

---

### 15. How do you properly implement equality for a custom class?

<details>
<summary>Reveal answer</summary>

You need to override four things for consistency:

```csharp
public class Money : IEquatable<Money>
{
    public decimal Amount { get; }
    public string Currency { get; }

    public override bool Equals(object? obj) => Equals(obj as Money);

    public bool Equals(Money? other) =>
        other is not null && Amount == other.Amount && Currency == other.Currency;

    public override int GetHashCode() => HashCode.Combine(Amount, Currency);

    public static bool operator ==(Money? a, Money? b) => Equals(a, b);
    public static bool operator !=(Money? a, Money? b) => !Equals(a, b);
}
```

If you skip `GetHashCode`, the object will behave incorrectly in `Dictionary` and `HashSet`. Consider using `record` which gives you all of this for free.

Deep dive: [Equals vs ==](../01-csharp-fundamentals/06-equals-vs-operator.md)

</details>

---

### 16. What is the difference between `const` and `readonly`?

<details>
<summary>Reveal answer</summary>

| Aspect | `const` | `readonly` |
|--------|---------|------------|
| Set when | Compile time | Runtime (constructor or declaration) |
| Stored | Inlined at call site | As a field in the object |
| Types allowed | Primitives, `string`, `null` | Any type |
| `static` | Implicitly static | Must add `static` explicitly |

```csharp
const double Pi = 3.14159;        // compile-time constant
readonly DateTime _created = DateTime.UtcNow; // runtime constant
```

**Gotcha**: if assembly A uses a `const` from assembly B, the value is baked into A at compile time. Changing the value in B requires **recompiling A** too.

Deep dive: [Modifiers](../01-csharp-fundamentals/05-modifiers.md)

</details>

---

### 17. How does the `switch` expression differ from the `switch` statement? When would you use each?

<details>
<summary>Reveal answer</summary>

The **switch statement** is imperative — each `case` executes statements. The **switch expression** (C# 8+) is functional — it returns a value.

```csharp
// Switch expression — concise, exhaustive
string label = status switch
{
    Status.Active   => "Active",
    Status.Inactive => "Inactive",
    _               => "Unknown",
};

// Switch statement — for side effects, multiple statements per case
switch (status)
{
    case Status.Active:
        logger.Log("Active");
        Activate();
        break;
}
```

Use **switch expressions** when mapping a value to another value. Use **switch statements** when each case triggers complex side effects.

</details>

---

### 18. What are namespaces, and why do they matter in larger codebases?

<details>
<summary>Reveal answer</summary>

A **namespace** is a logical grouping of types. It prevents name collisions and gives callers a hint about where something lives. Nothing about a namespace is enforced at runtime — it's purely a compile-time / organization tool.

```csharp
// File-scoped namespace (C# 10+)
namespace MyApp.Orders;

public class OrderService { ... }
```

Practical points:
- **File-scoped** namespaces (C# 10) remove one indent level vs the block form.
- `global using` declarations (C# 10) let you declare common usings once per project in a single file.
- **Namespace ≠ assembly**. One assembly can contain many namespaces; one namespace can span several assemblies.
- Large projects typically mirror namespaces to folder structure — not required by the compiler, but tooling (Rider/VS refactorings) assumes it.

Deep dive: [Namespace](../01-csharp-fundamentals/02-namespace.md)

</details>

---

### 19. What is the difference between `IDisposable` and `IAsyncDisposable`?

<details>
<summary>Reveal answer</summary>

Both provide **deterministic cleanup** of resources the GC can't handle on its own (file handles, DB connections, sockets).

- **`IDisposable`** exposes `void Dispose()`. Cleanup is synchronous.
- **`IAsyncDisposable`** exposes `ValueTask DisposeAsync()`. Cleanup is async — for things like network flushes, DB connection closes that shouldn't block a thread.

```csharp
// Sync
using (var conn = new SqlConnection(connStr)) { ... }   // Dispose() on exit

// Async — use 'await using' (not just 'using')
await using var stream = File.OpenRead("data.txt");     // DisposeAsync() on exit
```

Rules of thumb:
- A type that does I/O during cleanup should implement `IAsyncDisposable` (and usually `IDisposable` too, for sync callers).
- If a type implements both, call `DisposeAsync` when possible — `using` on such a type falls back to the synchronous path.
- Always `await using` inside `async` methods; plain `using` on an `IAsyncDisposable` calls the sync `Dispose()` which may block.

Deep dive: [Memory Leaks](../04-memory-and-performance/04-memory-leak.md)

</details>

---

[Back to index](README.md)
