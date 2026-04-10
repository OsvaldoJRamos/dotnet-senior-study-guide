# Stack and Heap

In C#, memory is divided into two main areas: **Stack** and **Heap**.

## Stack

Used to store **value types** (e.g., `int`, `float`, `struct`) and **references** to objects. It is fast and automatically managed. Whenever a method is called, a new block of memory is allocated on the stack, and when the method finishes, the memory is freed.

```
int a = 10; // local variable

┌─────────┐
│ int a=10 │
└─────────┘
   Stack
```

## Heap

Used to store **reference types** (e.g., objects, arrays, strings). When objects are created using `new`, they are stored on the heap. This memory remains allocated until the Garbage Collector (GC) determines it is no longer in use.

```
Test a = new Test();

┌─────┐      ┌────────┐
│  a  │ ──── │ object │
└─────┘      └────────┘
 Stack          Heap
```

## Code example

```csharp
class Program
{
    static void Main(string[] args)
    {
        int number = 10;             // Stored in the stack
        Person person = new Person(); // Stored in the heap
        person.Name = "Alice";       // The reference to the object is on the stack, but the object is on the heap
    }
}

class Person
{
    public string Name;
}
```

In this example, the integer `number` is stored on the stack, but the `Person` object and the string `"Alice"` are stored on the heap.

## SOH and LOH

Objects are allocated in two areas of the heap:

1. **Small Object Heap (SOH):** Objects smaller than 85,000 bytes (approximately 83 KB)
2. **Large Object Heap (LOH):** Objects larger than 85,000 bytes

The LOH differs from the SOH because large objects are expensive to allocate and deallocate. Allocations on the LOH **are not compacted** by the GC, meaning memory fragmentation can occur, leading to inefficient memory usage over time.

### Tips for the LOH:
- Minimize the number of large objects you allocate
- Instead of allocating large arrays, break them into smaller arrays that fit in the SOH
- Strings are immutable in C#, so concatenating large strings can cause allocations on the LOH. Use `StringBuilder` to reduce memory usage

---

[Back to index](README.md) | [Next: Garbage Collector →](02-garbage-collector.md)
