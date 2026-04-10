# Delegates, Events, Func and Action

## Delegates

A delegate is a **type-safe pointer to a method**. It defines the signature (parameters and return type) that the method must have.

```csharp
// Delegate declaration
public delegate int Operation(int a, int b);

// Methods that match the signature
int Add(int a, int b) => a + b;
int Multiply(int a, int b) => a * b;

// Usage
Operation op = Add;
Console.WriteLine(op(3, 4)); // 7

op = Multiply;
Console.WriteLine(op(3, 4)); // 12
```

## Func, Action and Predicate

Ready-made generic delegates from .NET -- **avoid creating custom delegates**:

| Delegate | Description | Signature |
|----------|-------------|-----------|
| `Func<T, TResult>` | Has a return value | `TResult method(T param)` |
| `Action<T>` | No return value (void) | `void method(T param)` |
| `Predicate<T>` | Returns bool | `bool method(T param)` |

```csharp
// Func: receives int, returns string
Func<int, string> convert = number => $"Value: {number}";
Console.WriteLine(convert(42)); // "Value: 42"

// Action: receives string, returns nothing
Action<string> log = msg => Console.WriteLine($"[LOG] {msg}");
log("Starting..."); // "[LOG] Starting..."

// Predicate: receives int, returns bool
Predicate<int> isEven = n => n % 2 == 0;
Console.WriteLine(isEven(4)); // True

// Func with multiple parameters
Func<int, int, int> add = (a, b) => a + b;
```

### Practical usage with LINQ

```csharp
var numbers = new List<int> { 1, 2, 3, 4, 5 };

// Where receives Func<int, bool>
var evens = numbers.Where(n => n % 2 == 0);

// Select receives Func<int, string>
var texts = numbers.Select(n => $"Number {n}");

// ForEach receives Action<int>
numbers.ForEach(n => Console.WriteLine(n));
```

## Events

Events are an **encapsulation layer** over delegates. They implement the **Observer** pattern -- they allow objects to be notified when something happens.

```csharp
public class OrderService
{
    // Event declaration
    public event EventHandler<Order>? OrderCreated;

    public void CreateOrder(Order order)
    {
        // ... creation logic ...

        // Fires the event
        OrderCreated?.Invoke(this, order);
    }
}

// Subscribers
var service = new OrderService();

service.OrderCreated += (sender, order) =>
    Console.WriteLine($"Email sent for order {order.Id}");

service.OrderCreated += (sender, order) =>
    Console.WriteLine($"Log recorded for order {order.Id}");

service.CreateOrder(new Order(1));
// "Email sent for order 1"
// "Log recorded for order 1"
```

### Event vs Delegate -- why use event?

```csharp
// WITH public delegate: anyone can invoke or overwrite
public Action<string>? OnMessage;

// Problem 1: external code can fire
obj.OnMessage?.Invoke("fake!"); // anyone can invoke

// Problem 2: external code can overwrite all handlers
obj.OnMessage = newHandler; // erases all previous ones

// WITH event: only the owning class can invoke
public event Action<string>? OnMessage;

// obj.OnMessage?.Invoke("fake!"); // COMPILE ERROR
// obj.OnMessage = newHandler;     // COMPILE ERROR
obj.OnMessage += myHandler;        // OK, can only use += and -=
```

## EventHandler pattern

The pattern recommended by Microsoft:

```csharp
// Custom event arguments
public class OrderEventArgs : EventArgs
{
    public int OrderId { get; }
    public decimal Value { get; }

    public OrderEventArgs(int orderId, decimal value)
    {
        OrderId = orderId;
        Value = value;
    }
}

// Class that publishes the event
public class OrderService
{
    public event EventHandler<OrderEventArgs>? OrderCreated;

    protected virtual void OnOrderCreated(OrderEventArgs e)
    {
        OrderCreated?.Invoke(this, e);
    }
}
```

## Delegates as Strategy Pattern

```csharp
public class Validator
{
    private readonly List<Func<string, bool>> _rules = new();

    public void AddRule(Func<string, bool> rule) => _rules.Add(rule);

    public bool Validate(string value) => _rules.All(rule => rule(value));
}

var validator = new Validator();
validator.AddRule(s => !string.IsNullOrEmpty(s));
validator.AddRule(s => s.Length >= 3);
validator.AddRule(s => s.All(char.IsLetterOrDigit));

Console.WriteLine(validator.Validate("abc123")); // True
Console.WriteLine(validator.Validate("ab"));      // False
```

## Summary

| Concept | When to use |
|---------|-------------|
| `Func<T>` | Callback with return value (LINQ, strategies) |
| `Action<T>` | Callback without return value (logging, side effects) |
| `event` | Notifications (Observer pattern) |
| Custom delegate | Rarely -- prefer Func/Action |

---

[← Previous: Generics](07-generics.md) | [Back to index](README.md)
