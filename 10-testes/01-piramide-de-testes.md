# Software Testing

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

Tests **must not depend on external services**. In CI/CD, tests run before the build. If the external service fails or does not have the required data, the tests fail and consequently the build fails.

## Unit Tests

Test the **smallest portion of the code** (public method, value object, etc.).

- Done at **larger scale** because they are simple to implement
- Use **mocks** for external dependencies
- Should be written by the **developer**
- Generally test **only public methods**

> If you need to test private methods, perhaps the class has **too much responsibility**. Consider moving the method to another class and making it public.

```csharp
[TestClass]
public class CalculadoraTests
{
    [TestMethod]
    public void Somar_DeveRetornarSomaCorreta()
    {
        var calc = new Calculadora();
        var resultado = calc.Somar(2, 3);
        Assert.AreEqual(5, resultado);
    }

    [TestMethod]
    [DataRow(0, 0, 0)]
    [DataRow(1, 1, 2)]
    [DataRow(-1, 1, 0)]
    public void Somar_ComDynamicData(int a, int b, int esperado)
    {
        var calc = new Calculadora();
        Assert.AreEqual(esperado, calc.Somar(a, b));
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
public class ClienteControllerTests
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
    public async Task Get_DeveRetornarClientes()
    {
        var response = await _client.GetAsync("/api/clientes");
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

[Next: Mocking and Best Practices →](02-mocking-e-boas-praticas.md) | [Back to index](README.md)
