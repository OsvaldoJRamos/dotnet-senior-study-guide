# C# Fundamentals

> Read the questions, think about your answer, then click to reveal.

---

### 1. What is the difference between value types and reference types in C#?

<details>
<summary>Reveal answer</summary>

- **Value types** (`int`, `bool`, `struct`, `enum`) are stored on the **stack** (or inline in the containing object). Assignment copies the value.
- **Reference types** (`class`, `interface`, `string`, `delegate`) are stored on the **heap**. Assignment copies the reference, not the object.

```csharp
int a = 10;
int b = a;   // b is an independent copy
b = 20;      // a is still 10

var list1 = new List<int> { 1 };
var list2 = list1;   // both point to the same object
list2.Add(2);        // list1 also has 2 elements now
```

Deep dive: [Stack and Heap](../03-memory-and-performance/01-stack-and-heap.md)

</details>

---

### 2. What is boxing and unboxing? When does it happen and why should you care?

<details>
<summary>Reveal answer</summary>

- **Boxing**: wrapping a value type in an `object` (allocates on the heap).
- **Unboxing**: extracting the value type back from the `object`.

```csharp
int x = 42;
object boxed = x;        // boxing — heap allocation
int y = (int)boxed;      // unboxing — type check + copy
```

It matters because boxing causes **heap allocations** and **GC pressure**. Common traps: using value types with non-generic collections (`ArrayList`), string interpolation before C# 10 with value types, and `Equals(object)` on structs.

Deep dive: [Stack and Heap](../03-memory-and-performance/01-stack-and-heap.md)

</details>

---

### 3. What is the difference between `Equals()` and `==` in C#?

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

### 4. Explain generic constraints. What is covariance and contravariance?

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

### 5. What are delegates, Func, Action, and events? How do they relate?

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

### 6. What is the difference between a record, a class, and a struct?

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

### 7. Why are strings immutable in C#? When should you use StringBuilder?

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

### 8. What are nullable reference types and why were they introduced?

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

### 9. How does pattern matching work in C#? Give practical examples.

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

### 10. What happens under the hood when you use async/await?

<details>
<summary>Reveal answer</summary>

The compiler transforms the `async` method into a **state machine** (a struct that implements `IAsyncStateMachine`). Each `await` becomes a state transition:

1. The method runs synchronously until the first `await` on an incomplete `Task`.
2. The state machine captures the current state and registers a continuation.
3. The thread is **released** (no blocking).
4. When the awaited task completes, the continuation resumes on the captured `SynchronizationContext` (or thread pool if `ConfigureAwait(false)`).

The method does **not** create a new thread — it just frees the current one to do other work. This is what makes async I/O scalable.

Deep dive: [Task, Async/Await](../04-concurrency-and-parallelism/03-task-async-await.md)

</details>

---

### 11. Explain IDisposable and the `using` statement. When do you need them?

<details>
<summary>Reveal answer</summary>

`IDisposable` provides a **deterministic cleanup** mechanism for unmanaged resources (file handles, DB connections, sockets). The `using` statement guarantees `Dispose()` is called even if an exception occurs.

```csharp
// Classic
using (var conn = new SqlConnection(connStr))
{
    // use connection
} // Dispose() called here

// C# 8+ using declaration
using var stream = File.OpenRead("data.txt");
// Dispose() called when the variable goes out of scope
```

You need `IDisposable` when your class holds unmanaged resources or wraps other disposables. If you don't dispose, you risk **memory leaks**, **connection pool exhaustion**, and **file locks**.

Deep dive: [Memory Leaks](../03-memory-and-performance/04-memory-leak.md)

</details>

---

### 12. What are `ref`, `out`, and `in` parameters?

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

### 13. What is the difference between an interface and an abstract class?

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

### 14. How do you properly implement equality for a custom class?

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

### 15. What is the difference between `const` and `readonly`?

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

### 16. How does the `switch` expression differ from the `switch` statement? When would you use each?

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

[Back to index](README.md)
