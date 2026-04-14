# Yield Return

## What it is

`yield return` allows you to create iterators that return elements **one at a time**, on demand, without needing to create a complete list in memory.

## Comparison: without yield vs with yield

### Without yield (creates the entire list in memory):
```csharp
List<int> tempResult = new();
foreach(var item in collection)
{
    tempResult.Add(item);
}
return tempResult;
```

### With yield (returns element by element):
```csharp
foreach(var item in collection)
{
    yield return item;
}
```

## When to use

Using `yield return` is recommended when **you won't need to use the entire return list**. If the function's consumer needs to use all items in the list, its use may not be recommended, as it can worsen performance.

## Example 1: Resource savings

### Without yield — loads everything into memory:
```csharp
// Returns one million items that can be iterated
List<object> GetAllItems()
{
    List<object> millionCustomers;
    database.LoadMillionCustomerRecords(millionCustomers);
    return millionCustomers;
}

// Call
int num = 0;
foreach(var itm in GetAllItems())
{
    num++;
    if (num == 5)
        break;
}
// Note: one million items are returned from the database, but only 5 are used.
```

### With yield — loads on demand:
```csharp
// Returns each item lazily; the source is itself streamed (e.g., IEnumerable<Customer>)
IEnumerable<Customer> IterateOverItems()
{
    foreach (var customer in database.Customers)
        yield return customer;
}

// Call
int num = 0;
foreach(var itm in IterateOverItems())
{
    num++;
    if (num == 5)
        break;
}
// Only 5 items are pulled from the source; the remaining ~999,995 are never materialized
```

The contrast is **eager vs lazy**: the first version forces the entire result set into memory before returning, while the yield version streams items one at a time and stops as soon as the consumer stops asking. For this scenario, using yield would be **much more performant**.

## Example 2: Responsive interface

Imagine a list being returned by a program with a user interface. Without yield, all data needs to be returned before it can be displayed in the interface. This can take a few seconds, or in extreme cases, even minutes. The user would think something went wrong.

With yield, you can get elements one by one and start displaying them to the user without blocking the interface.

## yield break

Used to terminate the iteration early:

```csharp
IEnumerable<int> EvenNumbers(int max)
{
    for (int i = 0; i <= max; i++)
    {
        if (i > 100)
            yield break; // stops generating elements
        
        if (i % 2 == 0)
            yield return i;
    }
}
```

---

[← Previous: ICollection and IList](02-icollection-ilist.md) | [Back to index](README.md) | [Next: Collections Overview →](04-collections-overview.md)
