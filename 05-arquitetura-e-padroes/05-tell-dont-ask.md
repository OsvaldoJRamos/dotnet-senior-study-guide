# Tell, Don't Ask

## Resumo

O princípio "Tell, Don't Ask" nos ajuda a estruturar o código de forma que os próprios objetos sejam responsáveis por suas operações, em vez de expor dados para que outra classe tome decisões. Isso melhora a coesão, reduz o acoplamento e torna o código mais alinhado com os princípios da Programação Orientada a Objetos.

## Exemplo

Temos uma classe de ContaBancaria com uma propriedade SALDO. Ao invés de existir um ContaBancariaService que pega o valor de saldo (ContaBancaria.Saldo) e valida se pode realizar o saque, essa lógica deve estar dentro da entidade.

### Errado (Ask):
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

### Correto (Tell):
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

A lógica de negócio (validação de saldo e limite) fica **dentro da entidade**, não em um serviço externo.

---

[← Anterior: Object Calisthenics](04-object-calisthenics.md) | [Voltar ao índice](README.md) | [Próximo: SAGA Pattern →](06-saga-pattern.md)
