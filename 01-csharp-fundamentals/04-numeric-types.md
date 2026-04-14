# Numeric Types: Float vs Double vs Decimal

## Comparison

| Type | Range | Precision (Digits) | Largest Exact Integer | Size |
|---|---|---|---|---|
| `float` | 1.5x10^-45 / 3.4x10^38 | 7 | 2^24 | 4 bytes |
| `double` | 5.0x10^-324 / 1.7x10^308 | 15-16 | 2^53 | 8 bytes |
| `decimal` | 1.0x10^-28 / 7.9x10^28 | 28-29 | 2^96 - 1 (~7.9x10^28) | 16 bytes |

## How they are stored in memory

When using `double` and `float`, the number's value is stored in memory using a type of "scientific notation" (like in math, 13x10^3) to save space. By using this approach to store values, **some decimals cannot be represented exactly in memory**.

This loss happens because `float` and `double` are **base-2** (binary), so common base-10 fractions like `0.1` have no finite binary representation; `decimal` uses **base-10** internally and avoids this issue.

## When to use each one

| Type | When to use | Example |
|---|---|---|
| `float` | Graphics, games, where precision is not as important | 3D coordinates, game physics |
| `double` | When you need more precision than float, scientific calculations | General mathematical calculations |
| `decimal` | Financial and monetary calculations, where precision is critical | Prices, salaries, taxes |

## Practical example

```csharp
// Precision problem with double
double a = 0.1 + 0.2;
Console.WriteLine(a == 0.3); // False! (result: 0.30000000000000004)

// Decimal is precise for monetary values
decimal b = 0.1m + 0.2m;
Console.WriteLine(b == 0.3m); // True
```

## Literal suffixes

```csharp
float f = 3.14f;     // suffix 'f'
double d = 3.14;     // default, no suffix (or 'd')
decimal m = 3.14m;   // suffix 'm'
```

---

[← Previous: CLR and IL](03-clr-and-il.md) | [Back to index](README.md) | [Next: Modifiers →](05-modifiers.md)
