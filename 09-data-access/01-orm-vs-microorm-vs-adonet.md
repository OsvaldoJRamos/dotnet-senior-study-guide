# ORM, Micro ORM or Pure ADO.NET

## Summary: when to use each one

| Criterion | ORM (EF Core) | Micro ORM (Dapper) | Pure ADO.NET |
|---|---|---|---|
| Productivity | High | Medium | Low |
| Performance | Medium | High | Maximum |
| SQL Control | Low | High | Total |
| Migrations | Yes | No | No |
| Testability | Good | Good | Difficult |
| Long-term maintenance | High | Medium | Low |
| Learning curve | Medium | Low | High |

### Practical conclusion:
- **EF Core (ORM):** ideal for 80% of commercial systems, CRUDs, REST APIs, enterprise applications
- **Dapper (Micro ORM):** ideal for performance-critical systems, heavy reporting, microservices, total control
- **Pure ADO.NET:** last resort; use only when you truly need low-level control

---

## ORM (Entity Framework Core)

### What is it?
A tool that maps C# objects to database tables, hiding SQL details. It allows you to work with the database as if you were working with objects.

### When to use?
- Medium to large projects with many entities
- You need productivity and minimal SQL code repetition
- Complex business rules, but without extreme performance requirements

### Advantages:
- High productivity (almost automatic CRUD)
- Built-in migrations
- Lazy/eager loading, tracking, LINQ
- Good integration with ASP.NET Core and DI

### Disadvantages:
- Performance overhead
- Little control over generated queries
- Complexity in advanced optimizations

### Real-world uses:
- Administrative systems
- RESTful APIs
- Applications with rich models and business rules

---

## Micro ORM (Dapper, RepoDb)

### What is it?
A lightweight tool that maps objects quickly, but requires you to write the SQL queries.

### When to use?
- Projects where performance is critical
- You prefer writing SQL manually
- You need more control and agility
- Systems with large data volumes or integrations with legacy databases

### Advantages:
- Super fast
- Simple to use
- Highly controlled
- Can be used alongside pure ADO.NET

### Disadvantages:
- No migrations
- No object tracking
- Less productivity in large projects

### Real-world uses:
- High-performance APIs
- Complex reports
- Services that perform massive database access
- Microservices with a specific database

---

## Pure ADO.NET

### What is it?
It is the foundation of everything: classes like `SqlConnection`, `SqlCommand`, `SqlDataReader`, etc. It allows total control over database access.

### When to use?
- You need maximum performance
- Extremely complex queries, manual tuning
- Integration with non-relational databases
- Legacy projects that already use ADO.NET
- Scenarios with low-level abstraction (e.g.: games, embedded devices)

### Advantages:
- Maximum control
- Zero abstraction
- No magic behind the scenes

### Disadvantages:
- Very verbose
- Low productivity
- Easy to make mistakes
- Requires manual SQL and mapping maintenance

### Example:
```csharp
var customers = new List<Customer>();

using var conn = new SqlConnection(connectionString);
conn.Open();

var cmd = new SqlCommand("SELECT * FROM Customers WHERE Active = 1", conn);
using var reader = cmd.ExecuteReader();

while (reader.Read())
{
    customers.Add(new Customer
    {
        Id = (int)reader["Id"],
        Name = reader["Name"].ToString()
    });
}
```

### Real-world uses:
- Embedded systems
- Internal tools focused on performance
- Applications with advanced tuning (caching, indexes, locks)

---

[Back to index](README.md) | [Next: Entity Framework →](02-entity-framework.md)
