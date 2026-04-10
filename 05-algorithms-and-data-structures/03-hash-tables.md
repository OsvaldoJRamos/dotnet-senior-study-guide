# Hash Tables

## How hash tables work

A hash table stores key-value pairs using a **hash function** to compute an index into an internal array of "buckets."

1. **Hash:** Compute `hashCode = key.GetHashCode()`
2. **Index:** Map to a bucket: `index = hashCode % bucketCount`
3. **Store/Retrieve:** Place or find the entry in that bucket

When two keys map to the same bucket, a **collision** occurs. .NET handles collisions using chaining (linked list per bucket).

```
Key "Alice" → GetHashCode() → 4821736 → % 8 → bucket 0
Key "Bob"   → GetHashCode() → 7293841 → % 8 → bucket 1
Key "Carol" → GetHashCode() → 5509200 → % 8 → bucket 0  ← collision with Alice!
```

> **Key insight:** Hash tables trade **space** for **speed**. The hash function gives O(1) average access, but collisions degrade to O(n) in the worst case.

## Dictionary\<TKey, TValue\>

The most used hash table in .NET. Stores unique key-value pairs.

```csharp
var users = new Dictionary<int, string>
{
    [1] = "Alice",
    [2] = "Bob",
    [3] = "Carol"
};

// Add
users[4] = "Dave";                      // O(1) avg
users.Add(5, "Eve");                    // O(1) avg — throws if key exists

// Access
string name = users[1];                 // O(1) avg — throws if key missing
bool found = users.TryGetValue(6, out var value); // O(1) avg — safe

// Remove
users.Remove(3);                        // O(1) avg

// Check
bool hasKey = users.ContainsKey(1);     // O(1) avg
bool hasVal = users.ContainsValue("Bob"); // O(n) — scans all values!
```

| Operation | Average | Worst |
|-----------|---------|-------|
| `Add` | O(1) | O(n) |
| `Remove` | O(1) | O(n) |
| `TryGetValue` / indexer | O(1) | O(n) |
| `ContainsKey` | O(1) | O(n) |
| `ContainsValue` | O(n) | O(n) |

> **Pitfall:** `ContainsValue` is O(n) because values are not hashed. If you need fast value lookups, maintain a reverse dictionary.

## HashSet\<T\>

A collection of **unique elements** with no key-value pairs — just the keys.

```csharp
var tags = new HashSet<string> { "csharp", "dotnet", "algorithms" };

tags.Add("csharp");      // returns false — already exists
tags.Add("performance"); // returns true — added

bool has = tags.Contains("dotnet"); // O(1) avg

// Set operations
var other = new HashSet<string> { "dotnet", "azure", "devops" };
tags.UnionWith(other);          // tags now has all from both
tags.IntersectWith(other);      // tags now has only shared items
tags.ExceptWith(other);         // tags now has items NOT in other
tags.IsSubsetOf(other);         // bool
tags.IsSupersetOf(other);       // bool
```

| Operation | Average | Worst |
|-----------|---------|-------|
| `Add` | O(1) | O(n) |
| `Remove` | O(1) | O(n) |
| `Contains` | O(1) | O(n) |
| `UnionWith` | O(n + m) | O(n + m) |
| `IntersectWith` | O(n) | O(n) |

> **Practical tip:** Use `HashSet<T>` when you need fast `Contains` checks. Converting a `List<T>` to a `HashSet<T>` before a loop with `Contains` can change O(n²) to O(n).

## ConcurrentDictionary\<TKey, TValue\>

Thread-safe dictionary for concurrent scenarios. Uses fine-grained locking (lock striping) instead of locking the entire collection.

```csharp
var cache = new ConcurrentDictionary<string, int>();

// Atomic operations
cache.TryAdd("counter", 0);
cache.AddOrUpdate("counter", 1, (key, old) => old + 1);
int value = cache.GetOrAdd("total", key => ComputeExpensiveValue(key));

// Safe enumeration (snapshot)
foreach (var kvp in cache)
    Console.WriteLine($"{kvp.Key}: {kvp.Value}");
```

| Scenario | Use |
|----------|-----|
| Single-threaded | `Dictionary<K,V>` — less overhead |
| Multi-threaded reads, rare writes | `Dictionary<K,V>` with `ReaderWriterLockSlim` |
| Frequent concurrent reads and writes | `ConcurrentDictionary<K,V>` |

> **Pitfall:** The `valueFactory` in `GetOrAdd` and `updateValueFactory` in `AddOrUpdate` are **not atomic**. The factory may run multiple times for the same key under contention. Only one result is stored.

## GetHashCode and Equals contract

When using custom types as dictionary keys or in hash sets, you **must** override both `GetHashCode()` and `Equals()`.

### The contract

1. **If `a.Equals(b)` is true, then `a.GetHashCode()` must equal `b.GetHashCode()`**
2. If hash codes are equal, objects are **not necessarily** equal (collisions happen)
3. `GetHashCode()` must be **consistent** — same object must always return the same hash during its lifetime in the collection

```csharp
public class Product
{
    public int Id { get; init; }
    public string Name { get; init; } = "";

    public override bool Equals(object? obj)
    {
        return obj is Product other && Id == other.Id;
    }

    public override int GetHashCode()
    {
        return HashCode.Combine(Id);
    }
}

// Now safe to use as dictionary key
var inventory = new Dictionary<Product, int>
{
    [new Product { Id = 1, Name = "Laptop" }] = 10
};
```

### Using `HashCode.Combine` (modern .NET)

```csharp
public override int GetHashCode()
{
    return HashCode.Combine(FirstName, LastName, BirthDate);
}
```

### Records get it for free

```csharp
public record Product(int Id, string Name);

// Equals and GetHashCode are generated automatically based on all properties
var dict = new Dictionary<Product, int>
{
    [new Product(1, "Laptop")] = 10
};

bool found = dict.ContainsKey(new Product(1, "Laptop")); // true
```

> **Pitfall:** Never use mutable properties in `GetHashCode`. If a property changes after the object is added to a dictionary, the hash changes but the bucket doesn't — the object becomes "lost" in the collection.

## Custom equality comparers

You can provide custom hashing/equality logic without modifying the class itself:

```csharp
public class CaseInsensitiveComparer : IEqualityComparer<string>
{
    public bool Equals(string? x, string? y)
        => string.Equals(x, y, StringComparison.OrdinalIgnoreCase);

    public int GetHashCode(string obj)
        => obj.ToUpperInvariant().GetHashCode();
}

var dict = new Dictionary<string, int>(new CaseInsensitiveComparer());
dict["Hello"] = 1;
bool found = dict.ContainsKey("hello"); // true

// Shortcut — .NET provides built-in comparers
var dict2 = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
```

## FrozenDictionary and FrozenSet (.NET 8+)

Immutable, **read-optimized** hash collections. They take longer to construct but are faster to read.

```csharp
using System.Collections.Frozen;

var config = new Dictionary<string, string>
{
    ["Region"] = "us-east-1",
    ["Env"] = "production"
};

FrozenDictionary<string, string> frozen = config.ToFrozenDictionary();
string region = frozen["Region"]; // faster than Dictionary for reads
```

> **Use case:** Configuration lookups, static reference data, or any dictionary that's built once and read many times.

---

[← Previous: Arrays and Linked Lists](02-arrays-and-linked-lists.md) | [Next: Sorting Algorithms →](04-sorting-algorithms.md) | [Back to index](README.md)
