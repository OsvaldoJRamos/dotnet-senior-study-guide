# Delegates, Events, Func e Action

## Delegates

Um delegate e um **ponteiro type-safe para um metodo**. Define a assinatura (parametros e retorno) que o metodo deve ter.

```csharp
// Declaracao do delegate
public delegate int Operacao(int a, int b);

// Metodos que combinam com a assinatura
int Somar(int a, int b) => a + b;
int Multiplicar(int a, int b) => a * b;

// Uso
Operacao op = Somar;
Console.WriteLine(op(3, 4)); // 7

op = Multiplicar;
Console.WriteLine(op(3, 4)); // 12
```

## Func, Action e Predicate

Delegates genericos prontos do .NET — **evitam criar delegates customizados**:

| Delegate | Descricao | Assinatura |
|----------|-----------|------------|
| `Func<T, TResult>` | Tem retorno | `TResult metodo(T param)` |
| `Action<T>` | Sem retorno (void) | `void metodo(T param)` |
| `Predicate<T>` | Retorna bool | `bool metodo(T param)` |

```csharp
// Func: recebe int, retorna string
Func<int, string> converter = numero => $"Valor: {numero}";
Console.WriteLine(converter(42)); // "Valor: 42"

// Action: recebe string, nao retorna nada
Action<string> logar = msg => Console.WriteLine($"[LOG] {msg}");
logar("Iniciando..."); // "[LOG] Iniciando..."

// Predicate: recebe int, retorna bool
Predicate<int> ehPar = n => n % 2 == 0;
Console.WriteLine(ehPar(4)); // True

// Func com multiplos parametros
Func<int, int, int> somar = (a, b) => a + b;
```

### Uso pratico com LINQ

```csharp
var numeros = new List<int> { 1, 2, 3, 4, 5 };

// Where recebe Func<int, bool>
var pares = numeros.Where(n => n % 2 == 0);

// Select recebe Func<int, string>
var textos = numeros.Select(n => $"Numero {n}");

// ForEach recebe Action<int>
numeros.ForEach(n => Console.WriteLine(n));
```

## Events

Events sao uma camada de **encapsulamento** sobre delegates. Implementam o padrao **Observer** — permitem que objetos sejam notificados quando algo acontece.

```csharp
public class PedidoService
{
    // Declaracao do evento
    public event EventHandler<Pedido>? PedidoCriado;

    public void CriarPedido(Pedido pedido)
    {
        // ... logica de criacao ...

        // Dispara o evento
        PedidoCriado?.Invoke(this, pedido);
    }
}

// Assinantes
var service = new PedidoService();

service.PedidoCriado += (sender, pedido) =>
    Console.WriteLine($"Email enviado para pedido {pedido.Id}");

service.PedidoCriado += (sender, pedido) =>
    Console.WriteLine($"Log registrado para pedido {pedido.Id}");

service.CriarPedido(new Pedido(1));
// "Email enviado para pedido 1"
// "Log registrado para pedido 1"
```

### Event vs Delegate — por que usar event?

```csharp
// COM delegate publico: qualquer um pode invocar ou sobrescrever
public Action<string>? OnMensagem;

// Problema 1: codigo externo pode disparar
obj.OnMensagem?.Invoke("falso!"); // qualquer um invoca

// Problema 2: codigo externo pode sobrescrever todos os handlers
obj.OnMensagem = novoHandler; // apaga todos os anteriores

// COM event: so a classe dona pode invocar
public event Action<string>? OnMensagem;

// obj.OnMensagem?.Invoke("falso!"); // ERRO de compilacao
// obj.OnMensagem = novoHandler;     // ERRO de compilacao
obj.OnMensagem += meuHandler;        // OK, so pode += e -=
```

## EventHandler pattern

O padrao recomendado pela Microsoft:

```csharp
// Argumentos customizados
public class PedidoEventArgs : EventArgs
{
    public int PedidoId { get; }
    public decimal Valor { get; }

    public PedidoEventArgs(int pedidoId, decimal valor)
    {
        PedidoId = pedidoId;
        Valor = valor;
    }
}

// Classe que publica o evento
public class PedidoService
{
    public event EventHandler<PedidoEventArgs>? PedidoCriado;

    protected virtual void OnPedidoCriado(PedidoEventArgs e)
    {
        PedidoCriado?.Invoke(this, e);
    }
}
```

## Delegates como Strategy Pattern

```csharp
public class Validador
{
    private readonly List<Func<string, bool>> _regras = new();

    public void AdicionarRegra(Func<string, bool> regra) => _regras.Add(regra);

    public bool Validar(string valor) => _regras.All(regra => regra(valor));
}

var validador = new Validador();
validador.AdicionarRegra(s => !string.IsNullOrEmpty(s));
validador.AdicionarRegra(s => s.Length >= 3);
validador.AdicionarRegra(s => s.All(char.IsLetterOrDigit));

Console.WriteLine(validador.Validar("abc123")); // True
Console.WriteLine(validador.Validar("ab"));      // False
```

## Resumo

| Conceito | Quando usar |
|----------|-------------|
| `Func<T>` | Callback com retorno (LINQ, strategies) |
| `Action<T>` | Callback sem retorno (logging, side effects) |
| `event` | Notificacoes (Observer pattern) |
| Delegate custom | Raramente — prefira Func/Action |

---

[← Anterior: Generics](07-generics.md) | [Voltar ao índice](README.md)
