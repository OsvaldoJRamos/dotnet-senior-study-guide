# Structs vs Classes

## Diferenças fundamentais

| Característica | `struct` (Value Type) | `class` (Reference Type) |
|---|---|---|
| Onde é armazenado | Stack (geralmente) | Heap |
| Passagem por parâmetro | Cópia do valor | Cópia da referência |
| Herança | Não suporta | Suporta |
| Pode ser null | Não (a menos que `Nullable<T>`) | Sim |
| Garbage Collection | Não precisa | Precisa |
| Default constructor | Sempre existe (não pode ser removido) | Pode ser customizado |
| Performance | Melhor para objetos pequenos e frequentes | Melhor para objetos complexos |

## Quando usar struct

Structs são melhores em cenários com **muitos objetos pequenos e de vida curta**, onde evitar alocações no heap faz diferença significativa:

- Simulação de água com um grande array de vetores de velocidade
- Jogos de construção de cidades com muitos objetos de mesmo comportamento (como carros)
- Sistemas de partículas em tempo real
- Renderização de CPU usando um grande array de pixels

## Quando usar class

- Quando o objeto precisa de herança
- Quando o objeto é grande (mais de ~16 bytes)
- Quando precisa de identidade (duas instâncias com mesmos valores são coisas diferentes)
- Quando precisa ser nulo (`null`)
- Na grande maioria das situações do dia a dia

## Exemplo prático

```csharp
// Struct - bom para coordenadas (pequeno, sem identidade)
public struct Point
{
    public double X { get; }
    public double Y { get; }
    
    public Point(double x, double y) => (X, Y) = (x, y);
}

// Class - bom para entidades de negócio (identidade, herança)
public class Cliente
{
    public Guid Id { get; }
    public string Nome { get; set; }
    public string Email { get; set; }
    
    public Cliente(string nome, string email)
    {
        Id = Guid.NewGuid();
        Nome = nome;
        Email = email;
    }
}
```

## Records (C# 9+)

Para cenários onde você quer **imutabilidade** e **igualdade por valor** sem usar struct:

```csharp
// record class - alocado no heap mas com igualdade por valor
public record Endereco(string Rua, string Cidade, string Estado);

// record struct (C# 10) - alocado na stack com igualdade por valor
public readonly record struct Coordenada(double Lat, double Lon);
```

---

[← Anterior: Memory Leak](04-memory-leak.md) | [Voltar ao índice](README.md)
