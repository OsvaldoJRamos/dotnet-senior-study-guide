# Entity Framework

> **Avoid lazy loading as the default in web/API code paths** — it causes silent N+1 and breaks once outside the `DbContext` scope. Use it selectively where domain services conditionally access navigation properties and you understand the lifetime.

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

3. **Register in DI (modern approach):**
```csharp
// Program.cs / Startup
builder.Services.AddDbContext<AppDbContext>(options =>
    options
        .UseSqlServer(builder.Configuration.GetConnectionString("Default"))
        .UseLazyLoadingProxies());
```

> `OnConfiguring` still works and is occasionally used for design-time tooling or standalone contexts, but `AddDbContext<T>` in DI is the standard registration for ASP.NET Core apps (it manages scope, disposal, and pooling).

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

## Change Tracking

EF Core's change tracker is how `SaveChangesAsync` knows what to INSERT/UPDATE/DELETE.

- **Snapshot tracking (default):** when you query without `AsNoTracking()`, EF stores a **snapshot** of each entity's original values. On `SaveChangesAsync`, it **diffs** current values vs the snapshot and generates SQL only for changed columns.
- **Cost:** every tracked entity takes memory (snapshot + metadata). On large read sets (e.g., 10k rows just for display), tracking hurts latency, allocation, and GC pressure.
- **`AsNoTracking()`:** skips the snapshot entirely. Use it for any read-only query — lists, reports, API responses that won't be saved back.
- **`AsNoTrackingWithIdentityResolution()`:** no tracking, but still deduplicates entities in the result graph (useful when the same row appears multiple times via joins).

```csharp
// Read-only — no tracking, much faster
var products = await context.Products
    .AsNoTracking()
    .Where(p => p.IsActive)
    .ToListAsync();

// Tracked — needed when you intend to modify and save
var product = await context.Products.FirstAsync(p => p.Id == id);
product.Price = newPrice;
await context.SaveChangesAsync(); // EF diffs against the snapshot
```

> Rule of thumb: tracked by default when you **will** mutate and save; `AsNoTracking()` otherwise. You can make it global with `context.ChangeTracker.QueryTrackingBehavior = QueryTrackingBehavior.NoTracking;` and opt-in with `.AsTracking()`.

## Cartesian explosion and split queries

Multiple `Include` on **collections** multiply rows in the result set (cartesian product): 1 blog × 10 posts × 20 comments = 200 rows, with blog/post columns duplicated over and over. The SQL is huge and the network transfer balloons.

```csharp
// Single query — cartesian explosion
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Contributors)
    .ToListAsync();

// Split query — one SELECT per collection, EF stitches results in memory
var blogs = await context.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Contributors)
    .AsSplitQuery()
    .ToListAsync();
```

Trade-off: split queries send multiple round trips and lose transactional consistency between them (rows could change between queries). Single-query is fine with one collection; reach for `AsSplitQuery()` when you see duplicated parent rows in logs or when `Include` chains multiple collections.

## Best practices

- Prefer **Eager Loading** in most API scenarios
- Use **AsNoTracking()** for read queries (better performance):
```csharp
var products = context.Products.AsNoTracking().ToList();
```
- Avoid **Select N+1** — always check the generated queries in the log

---

[← Previous: ORM vs Micro ORM vs ADO.NET](01-orm-vs-microorm-vs-adonet.md) | [Back to index](README.md) | [Next: Databases →](03-databases.md)
