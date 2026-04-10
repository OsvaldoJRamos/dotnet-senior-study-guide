# Mocking and Testing Best Practices

## Arrange-Act-Assert (AAA)

Pattern for organizing tests:

```csharp
[TestMethod]
public async Task AprovarPedido_DeveAtualizarStatus()
{
    // Arrange — set up the scenario
    var pedido = new Pedido("cliente-1", 100m);
    var repo = new Mock<IPedidoRepository>();
    repo.Setup(r => r.ObterPorIdAsync(pedido.Id)).ReturnsAsync(pedido);
    var service = new PedidoService(repo.Object);

    // Act — execute the action
    await service.AprovarAsync(pedido.Id);

    // Assert — verify the result
    Assert.AreEqual(StatusPedido.Aprovado, pedido.Status);
    repo.Verify(r => r.SalvarAsync(pedido), Times.Once);
}
```

## Moq (mocking framework)

### Setup and return

```csharp
var mock = new Mock<IProdutoRepository>();

// Simple return
mock.Setup(r => r.ObterPorIdAsync(1)).ReturnsAsync(new Produto("Notebook"));

// Return with callback
mock.Setup(r => r.ObterPorIdAsync(It.IsAny<int>()))
    .ReturnsAsync((int id) => new Produto($"Produto-{id}"));

// Throw exception
mock.Setup(r => r.ObterPorIdAsync(-1))
    .ThrowsAsync(new NotFoundException());
```

### Matchers (It)

```csharp
It.IsAny<int>()                    // any int
It.Is<int>(x => x > 0)            // positive int
It.IsIn(1, 2, 3)                   // 1, 2, or 3
It.IsRegex("[A-Z]+")              // regex
```

### Verify (verify calls)

```csharp
// Verify it was called
mock.Verify(r => r.SalvarAsync(It.IsAny<Pedido>()), Times.Once);

// Verify it was NOT called
mock.Verify(r => r.RemoverAsync(It.IsAny<int>()), Times.Never);

// Verify call with specific parameter
mock.Verify(r => r.SalvarAsync(It.Is<Pedido>(p => p.Status == StatusPedido.Aprovado)));
```

## Naming Conventions

```csharp
// Pattern: Method_Scenario_ExpectedResult
[TestMethod]
public void Somar_NumerosPositivos_DeveRetornarSoma()
public void Somar_ComZero_DeveRetornarOutroNumero()
public void Aprovar_PedidoJaAprovado_DeveLancarException()
public void CriarUsuario_EmailDuplicado_DeveRetornarConflito()
```

## Testing exceptions

```csharp
[TestMethod]
[ExpectedException(typeof(DomainException))]
public void Aprovar_PedidoCancelado_DeveLancarException()
{
    var pedido = new Pedido("cliente-1", 100m);
    pedido.Cancelar();
    pedido.Aprovar(); // deve lancar
}

// Or with Assert (more control)
[TestMethod]
public void Aprovar_PedidoCancelado_DeveLancarException()
{
    var pedido = new Pedido("cliente-1", 100m);
    pedido.Cancelar();

    var ex = Assert.ThrowsException<DomainException>(() => pedido.Aprovar());
    Assert.AreEqual("Não é possível aprovar pedido cancelado", ex.Message);
}
```

## DataRow / DynamicData (data-driven tests)

```csharp
[TestMethod]
[DataRow(1, 1, 2)]
[DataRow(0, 0, 0)]
[DataRow(-1, 1, 0)]
[DataRow(int.MaxValue, 1, int.MinValue)] // overflow
public void Somar_ComVariosInputs(int a, int b, int esperado)
{
    Assert.AreEqual(esperado, Calculadora.Somar(a, b));
}

// DynamicData for complex scenarios
[TestMethod]
[DynamicData(nameof(CenariosDeDesconto), DynamicDataSourceType.Method)]
public void CalcularDesconto_ComVariosCenarios(Pedido pedido, decimal esperado)
{
    Assert.AreEqual(esperado, pedido.CalcularDesconto());
}

private static IEnumerable<object[]> CenariosDeDesconto()
{
    yield return new object[] { new Pedido(100m, TipoCliente.Regular), 0m };
    yield return new object[] { new Pedido(100m, TipoCliente.Premium), 10m };
    yield return new object[] { new Pedido(1000m, TipoCliente.Premium), 150m };
}
```

## Integration tests with WebApplicationFactory

```csharp
[TestClass]
public class PedidoApiTests
{
    private WebApplicationFactory<Program> _factory;
    private HttpClient _client;

    [TestInitialize]
    public void Setup()
    {
        _factory = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    // Substituir banco real por in-memory
                    services.RemoveAll<DbContextOptions<AppDbContext>>();
                    services.AddDbContext<AppDbContext>(options =>
                        options.UseInMemoryDatabase("TestDb"));
                });
            });
        _client = _factory.CreateClient();
    }

    [TestMethod]
    public async Task CriarPedido_DeveRetornar201()
    {
        var dto = new { ClienteId = "c1", Valor = 100m };
        var response = await _client.PostAsJsonAsync("/api/pedidos", dto);

        Assert.AreEqual(HttpStatusCode.Created, response.StatusCode);
    }
}
```

## Best practices

1. **Test behavior, not implementation** -- if the implementation changes but behavior stays the same, the test should not break
2. **One assert per test** (ideally) -- makes diagnosis easier
3. **Do not test private methods** -- test through the public interface
4. **Tests must be independent** -- do not depend on execution order
5. **Do not test the framework** -- do not test whether EF Core does INSERT correctly
6. **Use builders** to create complex test objects
7. **Avoid logic in tests** -- no if/else/for in tests

---

[← Previous: Testing Pyramid](01-testing-pyramid.md) | [Next: Stress and Load Testing →](03-stress-and-load-testing.md) | [Back to index](README.md)
