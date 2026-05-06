# 02 - Exceptions

## Contents

1. [Fundamentals](01-fundamentals.md) - `try`/`catch`/`finally`, `throw`, exception properties
2. [Exception Hierarchy](02-exception-hierarchy.md) - `Exception` base, `SystemException` vs `ApplicationException`, common derived types
3. [Best Practices](03-best-practices.md) - When to catch, fail-fast, Tester-Doer / Try-Parse, what NOT to do
4. [Custom Exceptions](04-custom-exceptions.md) - Naming, three constructors, additional properties, when to create one
5. [Exception Filters (`when`)](05-exception-filters.md) - Filter semantics, no-stack-unwind advantage, debugging benefits
6. [`ExceptionDispatchInfo`](06-dispatch-info.md) - Capturing and rethrowing exceptions across boundaries with stack preservation
7. [Async Exceptions and `AggregateException`](07-async-and-aggregate.md) - Propagation through `await`, `Task.Exception`, `Flatten`, `Handle`
8. [Performance](08-performance.md) - Throw cost, Try-Parse pattern, when exceptions hurt

---

[Back to index](../README.md)
