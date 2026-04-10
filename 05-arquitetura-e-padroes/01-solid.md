# SOLID

SOLID é um acrônimo para cinco princípios de design orientado a objetos que ajudam a criar software mais manutenível, flexível e escalável.

## S - Single Responsibility Principle (SRP)

**Uma classe deve ter apenas um motivo para mudar.**

Cada classe deve ter uma única responsabilidade. Se uma classe faz muitas coisas, ela se torna difícil de manter.

```csharp
// ERRADO - classe com múltiplas responsabilidades
public class Pedido
{
    public void CalcularTotal() { }
    public void SalvarNoBanco() { }
    public void EnviarEmail() { }
}

// CORRETO - cada classe com uma responsabilidade
public class Pedido
{
    public decimal CalcularTotal() { ... }
}

public class PedidoRepository
{
    public void Salvar(Pedido pedido) { ... }
}

public class NotificacaoService
{
    public void EnviarConfirmacao(Pedido pedido) { ... }
}
```

## O - Open/Closed Principle (OCP)

**Aberto para extensão, fechado para modificação.**

Você deve poder adicionar novos comportamentos sem alterar o código existente.

```csharp
// ERRADO - precisa modificar a classe para cada novo tipo
public class CalculadoraDesconto
{
    public decimal Calcular(string tipo, decimal valor)
    {
        if (tipo == "VIP") return valor * 0.2m;
        if (tipo == "Premium") return valor * 0.1m;
        return 0;
    }
}

// CORRETO - extensível via novas implementações
public interface IDesconto
{
    decimal Calcular(decimal valor);
}

public class DescontoVip : IDesconto
{
    public decimal Calcular(decimal valor) => valor * 0.2m;
}

public class DescontoPremium : IDesconto
{
    public decimal Calcular(decimal valor) => valor * 0.1m;
}
```

## L - Liskov Substitution Principle (LSP)

**Subclasses devem poder substituir suas classes base sem quebrar o programa.**

Se a classe B herda de A, então B deve funcionar em qualquer lugar que A funciona.

```csharp
// ERRADO - Pinguim herda de Ave mas não voa
public class Ave
{
    public virtual void Voar() { Console.WriteLine("Voando..."); }
}

public class Pinguim : Ave
{
    public override void Voar() { throw new Exception("Não voa!"); } // viola LSP
}

// CORRETO - separar contratos
public interface IAve { }
public interface IAveVoadora : IAve
{
    void Voar();
}

public class Aguia : IAveVoadora
{
    public void Voar() { Console.WriteLine("Voando..."); }
}

public class Pinguim : IAve { } // não implementa Voar
```

## I - Interface Segregation Principle (ISP)

**Nenhum cliente deve ser forçado a depender de métodos que não usa.**

Prefira várias interfaces pequenas e específicas a uma interface grande e genérica.

```csharp
// ERRADO - interface grande demais
public interface IWorker
{
    void Trabalhar();
    void Comer();
    void Dormir();
}

// CORRETO - interfaces segregadas
public interface ITrabalhador
{
    void Trabalhar();
}

public interface ISerVivo
{
    void Comer();
    void Dormir();
}
```

## D - Dependency Inversion Principle (DIP)

**Dependa de abstrações, não de implementações.**

Módulos de alto nível não devem depender de módulos de baixo nível. Ambos devem depender de abstrações.

```csharp
// ERRADO - depende da implementação concreta
public class PedidoService
{
    private readonly SqlPedidoRepository _repo = new SqlPedidoRepository();
}

// CORRETO - depende da abstração
public class PedidoService
{
    private readonly IPedidoRepository _repo;

    public PedidoService(IPedidoRepository repo)
    {
        _repo = repo;
    }
}
```

> O DIP é a base para a Injeção de Dependência (DI). Veja mais em [Dependency Injection](../06-aspnet-core/01-dependency-injection.md).

---

[Voltar ao índice](README.md) | [Próximo: Design Patterns →](02-design-patterns.md)
