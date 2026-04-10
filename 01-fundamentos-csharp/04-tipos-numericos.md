# Tipos Numéricos: Float vs Double vs Decimal

## Comparação

| Tipo | Intervalo | Precisão (Dígitos) | Maior Inteiro Exato | Tamanho |
|---|---|---|---|---|
| `float` | 1.5x10^-45 / 3.4x10^38 | 7 | 2^24 | 4 bytes |
| `double` | 5.0x10^-324 / 1.7x10^308 | 15-16 | 2^53 | 8 bytes |
| `decimal` | 1.0x10^-28 / 7.9x10^28 | 28-29 | 2^113 | 16 bytes |

## Como são armazenados na memória

Quando utilizamos `double` e `float`, o valor do número é salvo na memória utilizando um tipo de "notação científica" (tipo na matemática, 13x10^3) para economizar espaço. E utilizando essa abordagem para armazenar valores, **alguns decimais não conseguem ser escritos exatamente na memória**.

## Quando usar cada um

| Tipo | Quando usar | Exemplo |
|---|---|---|
| `float` | Gráficos, jogos, onde a precisão não é tão importante | Coordenadas 3D, física de jogos |
| `double` | Quando precisa de mais precisão que float, cálculos científicos | Cálculos matemáticos gerais |
| `decimal` | Cálculos financeiros e monetários, onde a precisão é crítica | Preços, salários, impostos |

## Exemplo prático

```csharp
// Problema de precisão com double
double a = 0.1 + 0.2;
Console.WriteLine(a == 0.3); // False! (resultado: 0.30000000000000004)

// Decimal é preciso para valores monetários
decimal b = 0.1m + 0.2m;
Console.WriteLine(b == 0.3m); // True
```

## Sufixos literais

```csharp
float f = 3.14f;     // sufixo 'f'
double d = 3.14;     // padrão, sem sufixo (ou 'd')
decimal m = 3.14m;   // sufixo 'm'
```

---

[← Anterior: CLR e IL](03-clr-e-il.md) | [Voltar ao índice](README.md) | [Próximo: Modificadores →](05-modificadores.md)
