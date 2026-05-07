# Object-Oriented Programming (OOP) in C#

C# is an object-oriented language. Every type — even `int`, `string`, your records — implicitly derives from `System.Object`, so every value carries `Equals`, `GetHashCode`, `ToString`, and `GetType`.

OOP rests on **four pillars** (Microsoft's wording): Abstraction, Encapsulation, Inheritance, Polymorphism.

| Pillar | Definition (MS Learn) | C# constructs |
|---|---|---|
| **Abstraction** | Modeling the relevant attributes and interactions of entities as classes to define an abstract representation of a system. | `interface`, `abstract class`, well-named types |
| **Encapsulation** | Hiding the internal state and functionality of an object and only allowing access through a public set of functions. | Access modifiers (`private`, `protected`), properties, `init`-only setters, `readonly` |
| **Inheritance** | Ability to create new abstractions based on existing abstractions. | `class Derived : Base`, `: base(...)` |
| **Polymorphism** | Ability to implement inherited properties or methods in different ways across multiple abstractions. | `virtual` / `override`, `abstract`, interfaces, generics |

> **Senior framing**: pillars are descriptive, not prescriptive. In an interview, lead with *what problem each one solves*, not the textbook list.

---

## 1. Abstraction

Abstraction is about **what** the object exposes, not **how** it does it. The clean way is to depend on a contract (interface or abstract base) rather than a concrete implementation.

```csharp
public interface IInvoiceRepository
{
    Task<Invoice?> GetByIdAsync(Guid id);
    Task SaveAsync(Invoice invoice);
}

public class InvoiceService(IInvoiceRepository repo)
{
    public Task<Invoice?> Get(Guid id) => repo.GetByIdAsync(id); // depends on the contract
}
```

Swap `SqlInvoiceRepository` for `MongoInvoiceRepository` without touching `InvoiceService`. That's abstraction earning its keep.

---

## 2. Encapsulation

Encapsulation is about **invariants**, not about putting `private` in front of every field. A class with a public property and a public setter that anyone can mutate is a public field with extra steps.

```csharp
public class Order
{
    private readonly List<OrderItem> _items = new();

    public IReadOnlyList<OrderItem> Items => _items;       // read-only view
    public decimal Total { get; private set; }
    public OrderStatus Status { get; private set; } = OrderStatus.Draft;

    public void AddItem(OrderItem item)
    {
        if (Status != OrderStatus.Draft)
            throw new InvalidOperationException("Cannot modify a confirmed order.");
        if (item.Quantity <= 0)
            throw new ArgumentException("Quantity must be positive.", nameof(item));

        _items.Add(item);
        Total += item.Price * item.Quantity;
    }

    public void Confirm()
    {
        if (_items.Count == 0)
            throw new InvalidOperationException("Order has no items.");
        Status = OrderStatus.Confirmed;
    }
}
```

Rules of thumb:

- Expose collections via `IReadOnlyList<T>` / `IReadOnlyCollection<T>`, never the backing `List<T>`.
- Setters should be `private` or `init` unless the caller really needs to set the value after construction.
- Validation belongs **in the domain method**, not in the controller or service.

---

## 3. Inheritance

A derived class implicitly receives all members of its base class **except constructors and finalizers** ([MS Learn — Inheritance](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/object-oriented/inheritance)).

```csharp
public class WorkItem
{
    public int Id { get; protected set; }
    public string Title { get; protected set; } = "";

    public WorkItem(string title) => Title = title;

    public override string ToString() => $"{Id} - {Title}";  // overrides Object.ToString
}

public class ChangeRequest : WorkItem
{
    public int OriginalItemId { get; }

    // Constructor chaining: the derived class MUST satisfy the base constructor
    public ChangeRequest(string title, int originalId) : base(title)
        => OriginalItemId = originalId;
}
```

### Key facts

| Fact | Detail |
|---|---|
| Single inheritance | A class has **one** direct base class. Use interfaces for "multiple inheritance of behavior". |
| Transitivity | `C : B : A` → `C` inherits from both `B` and `A`. |
| Implicit `Object` | Every class without an explicit base derives from `System.Object`. |
| Structs | Cannot inherit from another `struct` or `class`. They can implement interfaces. |
| Constructors | Not inherited. Derived class must call a base constructor (explicit `: base(...)` or implicit no-arg). |
| `: this(...)` | Constructor chaining inside the same class — call another overload, share initialization logic. |

### Constructor chaining gotcha

If the base class declares **any** constructor, the compiler stops generating the default parameterless one. The derived class must explicitly call a base constructor:

```csharp
public abstract class Shape
{
    protected Shape(string color) { /* ... */ }   // No default ctor any more
}

public class Square : Shape
{
    // public Square() { }                        // ❌ won't compile
    public Square(string color) : base(color) { } // ✅
}
```

---

## 4. Polymorphism

Two senses worth distinguishing:

| Kind | Mechanism in C# | Resolved when |
|---|---|---|
| **Subtype (runtime)** | `virtual` / `override`, abstract methods, interface dispatch | At runtime, by the actual object type |
| **Ad hoc** | Method overloading, operator overloading | At compile time, by static type |
| **Parametric** | Generics (`List<T>`, `Func<T, TResult>`) | At compile time / JIT, no boxing for value types |

### Runtime polymorphism — `virtual` / `override`

```csharp
public class Shape
{
    public virtual void Draw() => Console.WriteLine("Performing base class drawing tasks");
}

public class Circle : Shape
{
    public override void Draw()
    {
        Console.WriteLine("Drawing a circle");
        base.Draw();    // optional — call the base implementation
    }
}

List<Shape> shapes = [new Circle(), new Rectangle(), new Triangle()];
foreach (var s in shapes) s.Draw();   // dispatches to the runtime type
```

> **From the docs**: *"At run-time, when client code calls the method, the CLR looks up the run-time type of the object, and invokes that override of the virtual method."*

### Hide vs override (`new` modifier)

The `new` modifier **hides** instead of **overriding**. Dispatch then depends on the **declared (compile-time) type**, not the runtime type.

```csharp
public class Base   { public          void DoWork() => Console.WriteLine("Base"); }
public class Derived : Base { public new void DoWork() => Console.WriteLine("Derived"); }

Derived d = new();
Base    b = d;

d.DoWork();   // "Derived"   — declared type is Derived
b.DoWork();   // "Base"      — declared type is Base, even though the object is Derived
```

Compare with `override`: same code with `virtual`/`override` would print `"Derived"` both times.

> **Rule**: if you find yourself writing `new` to hide a member, you almost certainly meant `override`. The compiler emits CS0108 if `new` is missing — that's a smell, fix the design.

### `sealed override` — stop the virtual chain

```csharp
public class A     { public virtual  void DoWork() {} }
public class B : A { public override void DoWork() {} }
public class C : B { public sealed override void DoWork() {} } // no further override
public class D : C { public new      void DoWork() {} }        // can only hide, not override
```

`sealed override` lets the JIT devirtualize calls — measurable wins in hot paths.

---

## 5. Abstract classes

```csharp
public abstract class Vehicle
{
    protected string _brand;

    protected Vehicle(string brand) => _brand = brand;        // abstract classes CAN have constructors

    public string GetInfo() => $"This is a {_brand} vehicle.";    // shared behavior
    public virtual void StartEngine()                              // overridable default
        => Console.WriteLine($"{_brand} engine is starting...");

    public abstract void Move();                                   // forced override
    public abstract int MaxSpeed { get; }
}

public class Car : Vehicle
{
    public Car(string brand) : base(brand) { }
    public override void Move() => Console.WriteLine($"{_brand} car is driving.");
    public override int MaxSpeed => 200;
}
```

### Hard rules (from MS Learn)

- You **can't** instantiate an abstract class with `new`.
- An abstract class **can** contain abstract members **and** fully implemented members (constructors, fields, properties, methods).
- An abstract class **cannot** be `sealed` — the modifiers are contradictory.
- Any non-abstract derived class **must** implement every abstract member.
- An abstract method is implicitly `virtual`. You cannot mark it `static` or `virtual` explicitly in a `class` (you can in an `interface`).
- You **can** have an abstract class with **zero** abstract members — it just becomes a base you can't instantiate directly.

---

## 6. Why an abstract base class instead of a regular class?

This is the senior question. Five concrete reasons:

### 6.1. Prevent direct instantiation

Some types make no sense by themselves. `Animal`, `Vehicle`, `Repository<T>` — there is no such thing in the domain. Marking the base `abstract` is a **compile-time guarantee** the codebase will never produce a meaningless `new Animal()`.

### 6.2. Force derived classes to fill in behavior

An `abstract` member has no body. The compiler refuses to compile a non-abstract derived class that forgets to override it. With a plain `virtual` method, the derived class is *allowed* to ignore it and inherit a (potentially wrong) default.

```csharp
public abstract class PaymentMethod
{
    public abstract void Charge(decimal amount);   // every method MUST implement
}

public class Pix : PaymentMethod
{
    public override void Charge(decimal amount) { /* ... */ }
}

// public class Bitcoin : PaymentMethod { }    // ❌ compile error: must implement Charge
```

### 6.3. Share state and invariants between siblings

Interfaces can't hold instance fields. An abstract base can — and that's the right place for shared state with shared invariants:

```csharp
public abstract class EntityBase
{
    public Guid Id { get; }                = Guid.NewGuid();
    public DateTime CreatedAt { get; }     = DateTime.UtcNow;
    public DateTime? UpdatedAt { get; protected set; }

    protected void Touch() => UpdatedAt = DateTime.UtcNow;
}
```

Putting `Id` / `CreatedAt` in an interface means every entity reimplements them; in an abstract class, they live in one place.

### 6.4. Template Method pattern (controlled extension points)

The base class owns the **algorithm**; derived classes plug in **steps**. This is exactly the pattern MS uses in their OOP tutorial: `BankAccount.MakeWithdrawal` runs the workflow and calls a `protected virtual CheckWithdrawalLimit` that subclasses override.

```csharp
public abstract class ImportJob
{
    public async Task RunAsync(CancellationToken ct)
    {
        await OpenSourceAsync(ct);
        var rows = await ReadAsync(ct);          // abstract — each importer reads differently
        var validated = Validate(rows);          // shared, can be overridden
        await WriteAsync(validated, ct);         // abstract
        await CloseSourceAsync(ct);
    }

    protected abstract Task<IReadOnlyList<Row>> ReadAsync(CancellationToken ct);
    protected abstract Task WriteAsync(IReadOnlyList<Row> rows, CancellationToken ct);
    protected virtual IReadOnlyList<Row> Validate(IReadOnlyList<Row> rows) => rows;
    protected virtual Task OpenSourceAsync(CancellationToken ct)  => Task.CompletedTask;
    protected virtual Task CloseSourceAsync(CancellationToken ct) => Task.CompletedTask;
}
```

You can't replicate this cleanly with a plain class plus `virtual` — there's no way to force the derived class to provide `Read`/`Write`.

### 6.5. Cleaner intent at the boundary

`abstract class` in the type signature tells the reader: *"this is meant to be extended, never used directly"*. A regular class with virtual members says *"this may be extended, may not"* — much weaker contract.

> **When NOT to make it abstract**: if every behavior has a sensible default and you actually want consumers to use the base type directly, keep it concrete. `abstract` is a constraint on consumers, and constraints have a cost.

---

## 7. Interfaces

```csharp
public interface IRepository<T>
{
    Task<T?> GetAsync(Guid id);
    Task SaveAsync(T entity);
}
```

### What's allowed in an interface

| Member | Allowed | Notes |
|---|---|---|
| Methods | ✅ | Body optional — no body = abstract |
| Properties | ✅ | Auto-properties **not** allowed (no instance fields) |
| Indexers, events, operators | ✅ | |
| Constants | ✅ | |
| Static fields, methods, properties | ✅ | Since C# 8 |
| Default implementation | ✅ | Since C# 8 |
| `static abstract` / `static virtual` members | ✅ | Since C# 11 — used for generic math, see below |
| Instance fields | ❌ | No instance state in an interface |
| Constructors | ❌ | Static constructors are allowed, instance constructors are not |

### Default access

- Top-level interfaces are `internal` by default.
- Members **without** a body (abstract members) are implicitly `public` — you can't change that.
- Members **with** a default implementation are `private` by default; you can mark them `public` / `protected` / `internal`.

### Default interface methods (C# 8)

You can supply a body in an interface to give implementers a default. This is **not** a substitute for an abstract class — call sites only see the default through the interface type:

```csharp
public interface ILogger
{
    void Log(LogLevel level, string message);

    void Info(string message)  => Log(LogLevel.Info, message);   // default
    void Error(string message) => Log(LogLevel.Error, message);  // default
}

ILogger logger = new ConsoleLogger();
logger.Info("hello");   // ✅ uses the default

ConsoleLogger c = new();
// c.Info("hello");      // ❌ won't compile — Info is only visible via ILogger
```

Use case: **adding a new method to a published interface without breaking implementers**.

### Static abstract members (C# 11) — generic math

Lets you write generic code that calls static members (operators, parsers, identity values):

```csharp
public static T Sum<T>(IEnumerable<T> values) where T : INumber<T>
{
    T total = T.Zero;             // static abstract
    foreach (var v in values) total += v;   // operator+, also static abstract
    return total;
}
```

`int`, `double`, `decimal`, `BigInteger` all implement `INumber<T>`, so `Sum(new[] { 1, 2, 3 })` and `Sum(new[] { 1.1, 2.2 })` both work, with no boxing.

### Explicit interface implementation

When two interfaces declare the same method, or you want to hide an interface member from the class's public surface:

```csharp
public interface IReader { string Read(); }
public interface IWriter { string Read(); }   // same name, same signature

public class Stream : IReader, IWriter
{
    string IReader.Read() => "from reader";
    string IWriter.Read() => "from writer";
}

var s = new Stream();
// s.Read();                   // ❌ ambiguous, not even available
((IReader)s).Read();           // "from reader"
((IWriter)s).Read();           // "from writer"
```

---

## 8. Abstract class vs interface

| Aspect | `abstract class` | `interface` |
|---|---|---|
| Multiple inheritance | ❌ (one base) | ✅ (many) |
| Instance fields / state | ✅ | ❌ |
| Instance constructors | ✅ | ❌ |
| Default implementation | ✅ | ✅ (C# 8+) |
| Member access modifiers | Any | `public` for abstract members; `private` default for default impls |
| Forces "is-a" hierarchy | ✅ | ❌ — capability, no hierarchy implied |
| Versioning new methods | Add `virtual` with default | Add default method (C# 8+) |
| Static abstract members | ❌ on classes | ✅ (C# 11+) |
| Best at | Sharing state + invariants | Decoupling, capabilities, multiple roles |

**Practical rule**:

- *Is-a, with shared state* → `abstract class` (`Vehicle`, `EntityBase`).
- *Capability without hierarchy* → `interface` (`IDisposable`, `IEquatable<T>`, `INotifyPropertyChanged`).
- *Plugin point in a generic algorithm* → `abstract class` Template Method, or `interface` Strategy. Strategy composes better; template is tighter.

---

## 9. Inheritance vs composition

C# allows inheritance, but the well-worn rule **"prefer composition over inheritance"** still holds:

```csharp
// Inheritance — coupled to the whole hierarchy
public class EmailNotifier : Notifier { /* override SendAsync */ }

// Composition — swap a single dependency
public class OrderService(INotifier notifier)
{
    public Task ConfirmAsync(Order o) => notifier.SendAsync($"order {o.Id} confirmed");
}
```

Use inheritance when:

- The relationship is genuinely "is-a", not "uses-a".
- You're going to share **state**, not just behavior.
- The base class is designed for extension (Template Method, abstract).

Avoid inheritance for code reuse alone — extract a class and inject it.

---

## 10. Senior gotchas

### 10.1. `virtual` is **not** the default in C#

In Java every non-`final` method is virtual; in C# only methods marked `virtual` (or `abstract`, or `override` of one) participate in dispatch. Forgetting this is the most common source of "why isn't my override being called?"

### 10.2. Calling virtual from a constructor

```csharp
public class Base
{
    public Base() => Init();         // ⚠ dispatches to Derived.Init before Derived's ctor body runs
    public virtual void Init() { }
}

public class Derived : Base
{
    private readonly string _name;
    public Derived(string name) => _name = name;     // assignment runs AFTER base() returned
    public override void Init() => Console.WriteLine(_name.Length);   // ❌ NRE — _name is still null
}
```

Order of construction in C# (per MS Learn): instance fields zeroed → derived field initializers → base field initializers → base constructor body → derived constructor body. Field initializers are safe by step 4, but **anything you assign in the derived constructor body** is still missing when the base constructor calls a virtual. Don't call virtuals from constructors.

### 10.3. Liskov Substitution Principle (LSP)

A derived class must be substitutable for its base **without breaking the contract**. The classic counter-example:

```csharp
public class Rectangle { public virtual int Width { get; set; } public virtual int Height { get; set; } }

public class Square : Rectangle
{
    public override int Width  { set { base.Width = value; base.Height = value; } }
    public override int Height { set { base.Width = value; base.Height = value; } }
}

void TestArea(Rectangle r) { r.Width = 5; r.Height = 4; Debug.Assert(r.Width * r.Height == 20); }

TestArea(new Square());   // ❌ assertion fails: Width=4, Height=4
```

`Square : Rectangle` violates LSP because setting one dimension on a square mutates the other — the base contract didn't promise that. Real fix: don't model square as a rectangle.

### 10.4. Equality + hash code

If you override `Equals`, you **must** override `GetHashCode`. Otherwise `Dictionary`, `HashSet`, and LINQ `GroupBy` silently break — equal objects land in different buckets.

```csharp
public override bool Equals(object? obj) => obj is Money m && Amount == m.Amount && Currency == m.Currency;
public override int  GetHashCode()       => HashCode.Combine(Amount, Currency);
```

Or just use `record` and let the compiler do it.

### 10.5. `sealed` is a feature, not a smell

Marking a class `sealed` allows the JIT to **devirtualize** calls (turn an indirect call into a direct one) — measurable in tight loops. It also closes a hole in the type system: callers can't subclass and break invariants. Default to `sealed`; remove it when there's a real extension point.

### 10.6. `protected` can be wider than you think

`protected` lets **any** derived class — including ones in third-party assemblies — touch the member. If you need "derived class **and** my assembly only", use `private protected` (C# 7.2+).

### 10.7. Records

`record` (and `record struct`) are compiler sugar over a class/struct with **value-based equality**, `GetHashCode`, `ToString`, deconstruction, and `with`-expressions auto-generated. They support inheritance (between records) but the equality implementation requires every record in the hierarchy to participate properly — read [Records](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/types/records) before using them in deep hierarchies.

### 10.8. `struct` cannot inherit, only implement interfaces

`struct` always inherits `System.ValueType` (which inherits `Object`), and it implements interfaces. Treating a struct like a small class is the fastest way to introduce hidden boxing.

---

## 11. Putting it together — a real example

```csharp
public interface IPaymentProcessor
{
    Task<PaymentResult> ChargeAsync(Money amount, CancellationToken ct);
}

public abstract class PaymentProcessorBase(ILogger logger) : IPaymentProcessor
{
    protected ILogger Logger { get; } = logger;

    public async Task<PaymentResult> ChargeAsync(Money amount, CancellationToken ct)
    {
        if (amount.Value <= 0) throw new ArgumentException("Amount must be positive.");

        Logger.LogInformation("Charging {Amount}", amount);
        try
        {
            var result = await ChargeCoreAsync(amount, ct);    // each gateway plugs in here
            Logger.LogInformation("Charge result: {Status}", result.Status);
            return result;
        }
        catch (Exception ex)
        {
            Logger.LogError(ex, "Charge failed");
            throw;
        }
    }

    protected abstract Task<PaymentResult> ChargeCoreAsync(Money amount, CancellationToken ct);
}

public sealed class StripeProcessor(ILogger<StripeProcessor> log, StripeClient client)
    : PaymentProcessorBase(log)
{
    protected override Task<PaymentResult> ChargeCoreAsync(Money amount, CancellationToken ct)
        => client.ChargeAsync(amount, ct);
}
```

What's happening:

- **Interface** (`IPaymentProcessor`) — consumers depend on the contract, easy to mock.
- **Abstract base** — owns logging, validation, error handling (cross-cutting concerns).
- **`abstract` method** (`ChargeCoreAsync`) — every gateway must implement it.
- **`sealed` class** — `StripeProcessor` is the leaf; nobody should subclass it again.
- **Composition** — `StripeClient` is injected, not inherited.

That's idiomatic C# OOP at a senior level.

---

## Useful Links

- [Object-Oriented Programming in C# — MS Learn](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/tutorials/oop)
- [Inheritance — MS Learn](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/object-oriented/inheritance)
- [Polymorphism — MS Learn](https://learn.microsoft.com/en-us/dotnet/csharp/fundamentals/object-oriented/polymorphism)
- [`abstract` keyword — C# Reference](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/abstract)
- [`interface` keyword — C# Reference](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/interface)
- [Versioning with the Override and New Keywords](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/versioning-with-the-override-and-new-keywords)
- [Knowing When to Use Override and New Keywords](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/knowing-when-to-use-override-and-new-keywords)
- [Static abstract members in interfaces](https://learn.microsoft.com/en-us/dotnet/csharp/advanced-topics/interface-implementation/static-virtual-interface-members)

---

[← Previous: Delegates and Events](08-delegates-and-events.md) | [Back to index](README.md)
