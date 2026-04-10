# Mocking e Boas Praticas de Testes

## Arrange-Act-Assert (AAA)

Padrao para organizar testes:

```csharp
[TestMethod]
public async Task AprovarPedido_DeveAtualizarStatus()
{
    // Arrange — prepara o cenario
    var pedido = new Pedido("cliente-1", 100m);
    var repo = new Mock<IPedidoRepository>();
    repo.Setup(r => r.ObterPorIdAsync(pedido.Id)).ReturnsAsync(pedido);
    var service = new PedidoService(repo.Object);

    // Act — executa a acao
    await service.AprovarAsync(pedido.Id);

    // Assert — verifica o resultado
    Assert.AreEqual(StatusPedido.Aprovado, pedido.Status);
    repo.Verify(r => r.SalvarAsync(pedido), Times.Once);
}
```

## Moq (framework de mocking)

### Setup e retorno

```csharp
var mock = new Mock<IProdutoRepository>();

// Retorno simples
mock.Setup(r => r.ObterPorIdAsync(1)).ReturnsAsync(new Produto("Notebook"));

// Retorno com callback
mock.Setup(r => r.ObterPorIdAsync(It.IsAny<int>()))
    .ReturnsAsync((int id) => new Produto($"Produto-{id}"));

// Lanca excecao
mock.Setup(r => r.ObterPorIdAsync(-1))
    .ThrowsAsync(new NotFoundException());
```

### Matchers (It)

```csharp
It.IsAny<int>()                    // qualquer int
It.Is<int>(x => x > 0)            // int positivo
It.IsIn(1, 2, 3)                   // 1, 2 ou 3
It.IsRegex("[A-Z]+")              // regex
```

### Verify (verificar chamadas)

```csharp
// Verificar que foi chamado
mock.Verify(r => r.SalvarAsync(It.IsAny<Pedido>()), Times.Once);

// Verificar que NAO foi chamado
mock.Verify(r => r.RemoverAsync(It.IsAny<int>()), Times.Never);

// Verificar chamada com parametro especifico
mock.Verify(r => r.SalvarAsync(It.Is<Pedido>(p => p.Status == StatusPedido.Aprovado)));
```

## Naming Conventions

```csharp
// Padrao: Metodo_Cenario_ResultadoEsperado
[TestMethod]
public void Somar_NumerosPositivos_DeveRetornarSoma()
public void Somar_ComZero_DeveRetornarOutroNumero()
public void Aprovar_PedidoJaAprovado_DeveLancarException()
public void CriarUsuario_EmailDuplicado_DeveRetornarConflito()
```

## Teste de excecoes

```csharp
[TestMethod]
[ExpectedException(typeof(DomainException))]
public void Aprovar_PedidoCancelado_DeveLancarException()
{
    var pedido = new Pedido("cliente-1", 100m);
    pedido.Cancelar();
    pedido.Aprovar(); // deve lancar
}

// Ou com Assert (mais controle)
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

// DynamicData para cenarios complexos
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

## Testes de integracao com WebApplicationFactory

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

## Boas praticas

1. **Teste comportamento, nao implementacao** — se mudar a implementacao e o comportamento for o mesmo, o teste nao deve quebrar
2. **Um assert por teste** (idealmente) — facilita diagnostico
3. **Nao teste metodos privados** — teste pela interface publica
4. **Testes devem ser independentes** — nao depender de ordem de execucao
5. **Nao teste o framework** — nao teste se EF Core faz INSERT corretamente
6. **Use builders** para criar objetos complexos de teste
7. **Evite logica nos testes** — sem if/else/for nos testes

---

[← Anterior: Pirâmide de Testes](01-piramide-de-testes.md) | [Voltar ao índice](README.md)
