# Namespace

## O que é

Namespaces servem para duas coisas principais:

- **Divisão lógica** — organizar o código, agrupando classes, estruturas e outros afins
- **Evitar conflitos de nomes** — de classes, estruturas, delegates, etc.

## O que significa o namespace

Quando você cria classes e outros tipos dentro de um namespace, na verdade o namespace faz parte do nome daquela classe.

```csharp
namespace EspacoNomes { public class MinhaClasse { } }
```

Na verdade o nome completo dessa classe é `EspacoNomes.MinhaClasse`.

## Usando classes sem indicar o nome completo

O nome completo da classe pode ser reduzido para somente `MinhaClasse` quando se faz uso dos `using`s no topo do arquivo, ou no início do namespace:

```csharp
using EspacoNomes; // faz com que todas as classes dentro do namespace possam
// ser referidas somente pelo nome final da classe
```

Também é possível renomear completamente uma classe com um using:

```csharp
using NovoNome = EspacoNomes.MinhaClasse;
```

Agora pode-se fazer referência à `EspacoNomes.MinhaClasse` usando-se simplesmente `NovoNome`.

## Referência direta dentro do mesmo namespace

Quando um código está dentro de um namespace, ele pode fazer referência direta a tudo que está diretamente no mesmo namespace. Por exemplo, duas classes dentro do mesmo namespace podem se referir uma à outra usando apenas o nome final da classe.

## File-scoped namespaces (C# 10+)

A partir do C# 10, é possível declarar o namespace sem chaves, aplicando-o ao arquivo inteiro:

```csharp
namespace MeuProjeto.Services;

public class MeuServico
{
    // Todo o arquivo pertence ao namespace MeuProjeto.Services
}
```

Isso reduz a indentação e é o padrão recomendado em projetos modernos.

---

[← Anterior: Ecossistema .NET](01-ecossistema-dotnet.md) | [Voltar ao índice](README.md) | [Próximo: CLR e IL →](03-clr-e-il.md)
