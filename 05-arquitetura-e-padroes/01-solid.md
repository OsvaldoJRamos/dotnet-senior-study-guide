# SOLID

SOLID is an acronym for five object-oriented design principles that help create more maintainable, flexible, and scalable software.

## S - Single Responsibility Principle (SRP)

**A class should have only one reason to change.**

Each class should have a single responsibility. If a class does too many things, it becomes difficult to maintain.

```csharp
// WRONG - class with multiple responsibilities
public class Pedido
{
    public void CalcularTotal() { }
    public void SalvarNoBanco() { }
    public void EnviarEmail() { }
}

// CORRECT - each class with one responsibility
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

**Open for extension, closed for modification.**

You should be able to add new behaviors without changing existing code.

```csharp
// WRONG - need to modify the class for each new type
public class CalculadoraDesconto
{
    public decimal Calcular(string tipo, decimal valor)
    {
        if (tipo == "VIP") return valor * 0.2m;
        if (tipo == "Premium") return valor * 0.1m;
        return 0;
    }
}

// CORRECT - extensible via new implementations
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

**Subclasses should be able to replace their base classes without breaking the program.**

If class B inherits from A, then B should work anywhere that A works.

```csharp
// WRONG - Pinguim inherits from Ave but doesn't fly
public class Ave
{
    public virtual void Voar() { Console.WriteLine("Voando..."); }
}

public class Pinguim : Ave
{
    public override void Voar() { throw new Exception("Não voa!"); } // violates LSP
}

// CORRECT - separate contracts
public interface IAve { }
public interface IAveVoadora : IAve
{
    void Voar();
}

public class Aguia : IAveVoadora
{
    public void Voar() { Console.WriteLine("Voando..."); }
}

public class Pinguim : IAve { } // does not implement Voar
```

## I - Interface Segregation Principle (ISP)

**No client should be forced to depend on methods it does not use.**

Prefer several small, specific interfaces over one large, generic interface.

```csharp
// WRONG - interface too large
public interface IWorker
{
    void Trabalhar();
    void Comer();
    void Dormir();
}

// CORRECT - segregated interfaces
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

**Depend on abstractions, not on implementations.**

High-level modules should not depend on low-level modules. Both should depend on abstractions.

```csharp
// WRONG - depends on concrete implementation
public class PedidoService
{
    private readonly SqlPedidoRepository _repo = new SqlPedidoRepository();
}

// CORRECT - depends on abstraction
public class PedidoService
{
    private readonly IPedidoRepository _repo;

    public PedidoService(IPedidoRepository repo)
    {
        _repo = repo;
    }
}
```

> DIP is the foundation for Dependency Injection (DI). See more in [Dependency Injection](../06-aspnet-core/01-dependency-injection.md).

---

[Back to index](README.md) | [Next: Design Patterns →](02-design-patterns.md)
