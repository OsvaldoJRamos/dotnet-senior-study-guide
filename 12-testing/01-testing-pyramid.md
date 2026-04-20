# Testing Pyramid

## Testing Pyramid

```
        /\
       /  \      E2E (few, expensive, critical)
      /----\
     /      \    Integration (medium volume)
    /--------\
   /          \  Unit (many, fast, cheap)
  /____________\
```

## Important point

Tests **must not depend on external services**. In CI/CD, tests run **after the build** (you can't run compiled tests before compilation), but **before deploy**. If the external service fails or does not have the required data, the tests fail and consequently the pipeline blocks the deploy.

## Unit Tests

Test the **smallest portion of the code** (public method, value object, etc.).

- Done at **larger scale** because they are simple to implement
- Use **mocks** for external dependencies
- Should be written by the **developer**
- Generally test **only public methods**

> If you need to test private methods, perhaps the class has **too much responsibility**. Consider moving the method to another class and making it public.

```csharp
[TestClass]
public class CalculatorTests
{
    [TestMethod]
    public void Sum_ShouldReturnCorrectSum()
    {
        var calc = new Calculator();
        var result = calc.Sum(2, 3);
        Assert.AreEqual(5, result);
    }

    [TestMethod]
    [DataRow(0, 0, 0)]
    [DataRow(1, 1, 2)]
    [DataRow(-1, 1, 0)]
    public void Sum_WithDynamicData(int a, int b, int expected)
    {
        var calc = new Calculator();
        Assert.AreEqual(expected, calc.Sum(a, b));
    }
}
```

## Integration Tests

Test whether **different parts work correctly together**.

- More **costly** to write, so done in **smaller quantity**
- Harder to pinpoint the problems
- Do not always cover all scenarios

Examples:
- Controller calls
- Database reads and writes
- Reading from and writing to queues
- File system usage

```csharp
[TestClass]
public class CustomerControllerTests
{
    private WebApplicationFactory<Program> _factory;
    private HttpClient _client;

    [TestInitialize]
    public void Setup()
    {
        _factory = new WebApplicationFactory<Program>();
        _client = _factory.CreateClient();
    }

    [TestMethod]
    public async Task Get_ShouldReturnCustomers()
    {
        var response = await _client.GetAsync("/api/customers");
        response.EnsureSuccessStatusCode();
    }
}
```

## E2E Tests (End-to-End)

Simulate the **user's interaction** with the application.

- **Extremely difficult** to write and maintain
- Should only be present for the **most critical** parts
- Test the complete flow (frontend -> backend -> database)

Tools: Playwright, Selenium, Cypress

## Contract Tests

Verify that the **interface (contract)** between services is being respected.

Useful in microservices to ensure that changes in an API do not break consumers.

## Fixture lifetime

Different test frameworks instantiate and tear down fixtures differently. Senior devs should know the difference:

| Framework | Per-test setup | Per-class / shared setup |
|---|---|---|
| **xUnit** | New class instance per test (constructor = setup, `IDisposable.Dispose` = teardown) | `IClassFixture<T>` (shared within one class) / `ICollectionFixture<T>` (shared across classes in a collection) |
| **MSTest** | `[TestInitialize]` / `[TestCleanup]` | `[ClassInitialize]` / `[ClassCleanup]` (static methods) |
| **NUnit** | `[SetUp]` / `[TearDown]` | `[OneTimeSetUp]` / `[OneTimeTearDown]` |

> xUnit's "new instance per test" is the key distinction — it forces test isolation by default. In MSTest/NUnit, instance state leaks unless you reset it in `[TestInitialize]`/`[SetUp]`.

## Integration tests with real dependencies: TestContainers

**Testcontainers.NET** (`Testcontainers` NuGet) is the senior-standard approach for integration tests that need a real database, message broker, or other service. It spins up throwaway Docker containers per test run (Postgres, SQL Server, RabbitMQ, Redis, etc.), giving you real SQL semantics and real transactions — without the drawbacks of EF Core's InMemory provider or shared dev databases.

## Test Doubles

Objects that replace real dependencies in tests:

| Type | Description |
|------|-------------|
| **Mock** | Verifies that methods were called correctly |
| **Stub** | Returns pre-defined values |
| **Fake** | Simplified implementation (e.g., in-memory database) |
| **Spy** | Records calls for later verification |
| **Dummy** | Object that only fills a parameter, never actually used |

---

[Next: Mocking and Best Practices →](02-mocking-and-best-practices.md) | [Back to index](README.md)
