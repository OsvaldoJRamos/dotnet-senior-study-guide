# Testes de Software

## Piramide de Testes

```
        /\
       /  \      E2E (poucos, caros, criticos)
      /----\
     /      \    Integracao (medio volume)
    /--------\
   /          \  Unitarios (muitos, rapidos, baratos)
  /____________\
```

## Ponto importante

Testes **nao podem depender de servicos externos**. Em CI/CD, testes sao executados antes do build. Se o servico externo falhar ou nao tiver os dados necessarios, os testes falham e consequentemente o build.

## Testes Unitarios

Testam a **menor porcao do codigo** (metodo publico, value object, etc.).

- Feitos em **maior escala** por serem simples de implementar
- Usam **mocks** para dependencias externas
- Devem ser feitos pelo **desenvolvedor**
- Geralmente testam **somente metodos publicos**

> Se e necessario testar metodos privados, talvez a classe tenha **muita responsabilidade**. Considere mover o metodo para outra classe e torna-lo publico.

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

## Testes de Integracao

Testam se **partes diferentes funcionam corretamente juntas**.

- Mais **custosos** de fazer, por isso feitos em **menor quantidade**
- Mais dificil encontrar os problemas
- Nem sempre abrangem todos os cenarios

Exemplos:
- Chamadas de controllers
- Leitura e escrita em banco de dados
- Lendo e gravando em filas
- Utilizacao de sistemas de arquivo

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

## Testes E2E (Ponta a Ponta)

Simulam a **interacao do usuario** com a aplicacao.

- **Extremamente dificeis** de fazer e manter
- Devem estar presentes somente nas partes **mais criticas**
- Testam o fluxo completo (frontend -> backend -> banco)

Ferramentas: Playwright, Selenium, Cypress

## Contract Tests

Verificam que a **interface (contrato)** entre servicos esta sendo respeitada.

Util em microservices para garantir que mudancas em uma API nao quebrem os consumidores.

## Test Doubles

Objetos que substituem dependencias reais nos testes:

| Tipo | Descricao |
|------|-----------|
| **Mock** | Verifica se metodos foram chamados corretamente |
| **Stub** | Retorna valores pre-definidos |
| **Fake** | Implementacao simplificada (ex: banco em memoria) |
| **Spy** | Registra chamadas para verificacao posterior |
| **Dummy** | Objeto que so preenche parametro, nunca e usado |

---

[Próximo: Mocking e Boas Práticas →](02-mocking-e-boas-praticas.md) | [Voltar ao índice](README.md)
