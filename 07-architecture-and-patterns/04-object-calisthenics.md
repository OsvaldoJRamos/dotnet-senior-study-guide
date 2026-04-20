# Object Calisthenics

Object Calisthenics is a set of **object-oriented programming best practices** created by Jeff Bay. The idea is to train developers to write clean, modular, and easily maintainable code through **self-imposed constraints** that promote a more disciplined coding style.

These rules combine well with **Clean Code** principles by Robert C. Martin, as they force the code to be **simple, cohesive, readable, and testable**.

## The 9 Rules

### 1. Only One Level Of Indentation Per Method
**Avoid multiple levels of indentation**. Instead, **extract code into private methods** or use early returns.

### 2. Don't Use The ELSE Keyword
The `else` keyword can indicate unnecessary logic. Use **early return** to simplify the flow.

### 3. Wrap All Primitives And Strings
Avoid using primitive types directly when they have **business meaning**. Use value objects (`Value Object`).

### 4. First Class Collections
Avoid using lists, arrays, or dictionaries directly. Encapsulate them inside a class.

**Bad example:**
```csharp
public class Order
{
    public List<Item> Items { get; set; }
}
```

**Good example:**
```csharp
public class OrderItems
{
    private readonly List<Item> _items = new();

    public void Add(Item item)
    {
        _items.Add(item);
    }

    public IEnumerable<Item> All() => _items;
}

public class Order
{
    public OrderItems Items { get; }

    public Order(OrderItems items)
    {
        Items = items;
    }
}
```

### 5. One Dot Per Line
Avoid chaining too many method calls (Law of Demeter). Each line should have at most one "dot" of navigation.

### 6. Don't Abbreviate
Use descriptive names. Abbreviations make reading harder.

### 7. Keep All Entities Small
Keep classes and methods small and focused.

### 8. No Classes With More Than Two Instance Variables
Limiting instance variables forces decomposition into smaller and more cohesive objects.

**Bad example:**
```csharp
public class ReportGenerator
{
    private string _title;
    private string _author;
    private DateTime _created;
    private List<string> _sections;
    private string _footer;
}
```

**Good example -- split into smaller, more cohesive objects:**
```csharp
public class ReportMetadata
{
    public string Title { get; }
    public string Author { get; }
    public DateTime Created { get; }

    public ReportMetadata(string title, string author, DateTime created)
    {
        Title = title;
        Author = author;
        Created = created;
    }
}

public class Report
{
    private readonly ReportMetadata _metadata;
    private readonly List<string> _sections;

    public Report(ReportMetadata metadata, List<string> sections)
    {
        _metadata = metadata;
        _sections = sections;
    }
}
```

### 9. No Getters/Setters/Properties
Avoid exposing internal state. Use methods that perform actions (see [Tell, Don't Ask](05-tell-dont-ask.md)).

---

[← Previous: KISS, DRY and YAGNI](03-kiss-dry-yagni.md) | [Back to index](README.md) | [Next: Tell, Don't Ask →](05-tell-dont-ask.md)
