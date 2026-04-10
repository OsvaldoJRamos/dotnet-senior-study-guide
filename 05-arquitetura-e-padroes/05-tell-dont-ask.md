# Tell, Don't Ask

## Summary

The "Tell, Don't Ask" principle helps us structure code so that objects themselves are responsible for their operations, instead of exposing data for another class to make decisions. This improves cohesion, reduces coupling, and makes the code more aligned with Object-Oriented Programming principles.

## Example

We have a BankAccount class with a BALANCE property. Instead of having a BankAccountService that retrieves the balance value (BankAccount.Balance) and validates whether the withdrawal can be made, this logic should be inside the entity.

### Wrong (Ask):
```csharp
public class ContaBancariaService
{
    public void ProcessarSaque(ContaBancaria conta, decimal valor)
    {
        if (conta.Saldo >= valor)
        {
            conta.Sacar(valor);
        }
        else
        {
            Console.WriteLine("Saldo insuficiente!");
        }
    }
}
```

### Correct (Tell):
```csharp
public class ContaBancaria
{
    private decimal _saldo;
    private decimal _limiteDiario;
    private decimal _saqueHoje;

    public ContaBancaria(decimal saldoInicial, decimal limiteDiario)
    {
        _saldo = saldoInicial;
        _limiteDiario = limiteDiario;
        _saqueHoje = 0;
    }

    public void Sacar(decimal valor)
    {
        if (_saqueHoje + valor > _limiteDiario)
            throw new InvalidOperationException("Limite diário de saque excedido!");
        if (_saldo < valor)
            throw new InvalidOperationException("Saldo insuficiente!");
        _saldo -= valor;
        _saqueHoje += valor;
    }

    public decimal ObterSaldo() => _saldo;
}
```

The business logic (balance and limit validation) stays **inside the entity**, not in an external service.

---

[← Previous: Object Calisthenics](04-object-calisthenics.md) | [Back to index](README.md) | [Next: SAGA Pattern →](06-saga-pattern.md)
