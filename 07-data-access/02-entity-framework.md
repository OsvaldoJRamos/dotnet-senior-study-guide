# Entity Framework

> Generally lazy load makes too many queries and becomes a mess, performance drops significantly. **AVOID USING IT AS MUCH AS POSSIBLE.**

## Lazy Loading

**Definition:** Related data **is not loaded automatically** when you query the main entity. It is only fetched from the database **when accessed for the first time**.

### Example:
```csharp
var blog = context.Blogs.First();
var posts = blog.Posts; // Here EF makes a new query to load Posts
```

### Advantages:
- Avoids bringing unnecessary data
- Can improve performance in simple scenarios

### Disadvantages:
- Can cause the **N+1 queries** problem (many extra queries)
- Requires navigation properties to be `virtual`
- In high-load scenarios, it can create bottlenecks

### How to enable Lazy Loading:

1. **Virtual navigation properties:**
```csharp
public class Blog
{
    public int BlogId { get; set; }
    public string Name { get; set; }

    // Must be virtual
    public virtual ICollection<Post> Posts { get; set; }
}
```

2. **Proxies package (in EF Core):**
```bash
dotnet add package Microsoft.EntityFrameworkCore.Proxies
```

3. **Configure in DbContext:**
```csharp
protected override void OnConfiguring(DbContextOptionsBuilder optionsBuilder)
{
    optionsBuilder
        .UseSqlServer("your-connection-string")
        .UseLazyLoadingProxies();
}
```

---

## Eager Loading

**Definition:** Related data is **loaded together with the main query** using `Include`.

### Example:
```csharp
var blog = context.Blogs
    .Include(b => b.Posts)
    .First();
```

With `ThenInclude` for deeper relationships:
```csharp
var blogs = context.Blogs
    .Include(b => b.Posts)
        .ThenInclude(p => p.Comments)
    .ToList();
```

Here you load everything at once.

### Advantages:
- Reduces the number of queries (better against the N+1 problem)
- Good when you **already know** you will need the related data

### Disadvantages:
- May load too much data, consuming more memory and network traffic
- Queries can become heavier

---

## When to use each one?

| Scenario | Use |
|---|---|
| When you don't always need the related data | Lazy Loading |
| Simple scenarios, few relationships, low volume | Lazy Loading |
| When you know you will **always need** the related entities | Eager Loading |
| Queries for APIs that return complete objects (DTOs, ViewModels) | Eager Loading |

## Explicit Loading (alternative)

When you want full control over when to load related data:

```csharp
var blog = context.Blogs.First();

// Explicitly loads when needed
context.Entry(blog)
    .Collection(b => b.Posts)
    .Load();
```

## Best practices

- Prefer **Eager Loading** in most API scenarios
- Use **AsNoTracking()** for read queries (better performance):
```csharp
var products = context.Products.AsNoTracking().ToList();
```
- Avoid **Select N+1** — always check the generated queries in the log

---

[← Previous: ORM vs Micro ORM vs ADO.NET](01-orm-vs-microorm-vs-adonet.md) | [Back to index](README.md) | [Next: Databases →](03-databases.md)
