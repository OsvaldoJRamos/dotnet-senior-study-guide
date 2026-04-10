# Dependency Injection (DI), IoC, DIP e Service Locator

## DIP (Dependency Inversion Principle)

Princípio SOLID que prega que devemos **depender de abstrações e não de implementações**. Falando a grosso modo, seria depender da interface e não da classe. Isso reduz o **acoplamento** entre classes, facilitando testes, manutenção e reutilização.

> Veja mais em [SOLID](../05-arquitetura-e-padroes/01-solid.md)

## IoC (Inversion of Control)

É o **padrão** que diz: "não crie suas dependências, receba-as de fora". Em vez de a classe criar seus próprios objetos, alguém de fora fornece.

## DI (Dependency Injection)

É a **técnica** que implementa o Inversion of Control. Ou seja, quando falamos que estamos trabalhando com Dependency Injection, estamos na verdade aplicando o padrão de Inversion of Control.

Ao invés de instanciar coisas na minha classe, eu externalizo essa responsabilidade e injeto as dependências na minha classe. Ou seja, a minha classe deixa de ser responsável por criar e passa a ser dependente de uma implementação.

```csharp
// SEM DI - classe cria sua dependência
public class PedidoService
{
    private readonly PedidoRepository _repo = new PedidoRepository();
}

// COM DI - dependência é injetada
public class PedidoService
{
    private readonly IPedidoRepository _repo;

    public PedidoService(IPedidoRepository repo)
    {
        _repo = repo; // recebida de fora
    }
}
```

## Service Locator

Faz o "de para". Dada uma abstração X, vou utilizar a implementação Y. O ASP.NET Core já traz esse container como padrão, embora tenha outros containers que façam isso como o Ninject.

Para configurar isso devemos ir no startup e definir como singleton, scoped ou transient.

```csharp
// Program.cs / Startup.cs
builder.Services.AddScoped<IPedidoRepository, PedidoRepository>();
builder.Services.AddTransient<IEmailService, SmtpEmailService>();
builder.Services.AddSingleton<ICacheService, RedisCacheService>();
```

## Formas de injeção

### 1. Constructor Injection (recomendada)
```csharp
public class PedidoService
{
    private readonly IPedidoRepository _repo;
    public PedidoService(IPedidoRepository repo) => _repo = repo;
}
```

### 2. Method Injection
```csharp
public void Processar([FromServices] IPedidoRepository repo)
{
    // usado em controllers, minimal APIs
}
```

### 3. Primary Constructor (C# 12)
```csharp
public class PedidoService(IPedidoRepository repo)
{
    public void Criar() => repo.Salvar(new Pedido());
}
```

---

[Voltar ao índice](README.md) | [Próximo: Service Lifetimes →](02-service-lifetimes.md)
