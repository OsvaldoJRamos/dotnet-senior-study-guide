# Numeric Types: Float vs Double vs Decimal

## Comparison

| Type | Range | Precision (Digits) | Largest Exact Integer | Size |
|---|---|---|---|---|
| `float` | 1.5x10^-45 / 3.4x10^38 | 7 | 2^24 | 4 bytes |
| `double` | 5.0x10^-324 / 1.7x10^308 | 15-16 | 2^53 | 8 bytes |
| `decimal` | 1.0x10^-28 / 7.9x10^28 | 28-29 | 2^113 | 16 bytes |

## How they are stored in memory

When using `double` and `float`, the number's value is stored in memory using a type of "scientific notation" (like in math, 13x10^3) to save space. By using this approach to store values, **some decimals cannot be represented exactly in memory**.

## When to use each one

| Type | When to use | Example |
|---|---|---|
| `float` | Graphics, games, where precision is not as important | 3D coordinates, game physics |
| `double` | When you need more precision than float, scientific calculations | General mathematical calculations |
| `decimal` | Financial and monetary calculations, where precision is critical | Prices, salaries, taxes |

## Practical example

```csharp
// Problema de precisão com double
double a = 0.1 + 0.2;
Console.WriteLine(a == 0.3); // False! (resultado: 0.30000000000000004)

// Decimal é preciso para valores monetários
decimal b = 0.1m + 0.2m;
Console.WriteLine(b == 0.3m); // True
```

## Literal suffixes

```csharp
float f = 3.14f;     // sufixo 'f'
double d = 3.14;     // padrão, sem sufixo (ou 'd')
decimal m = 3.14m;   // sufixo 'm'
```

---

[← Previous: CLR and IL](03-clr-e-il.md) | [Back to index](README.md) | [Next: Modifiers →](05-modificadores.md)
