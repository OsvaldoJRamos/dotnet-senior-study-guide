# Modificadores em C#

## Modificadores de Acesso (Métodos e Membros)

### 1. `public`
- **Uso:** Acesso irrestrito. Pode ser usado em qualquer lugar.
- **Quando evitar:** Se não quiser expor detalhes internos da aplicação.

### 2. `private`
- **Uso:** Acesso limitado à própria classe.
- **Quando usar:** Para esconder detalhes internos, como variáveis auxiliares e métodos de implementação.

### 3. `protected`
- **Uso:** Acesso dentro da própria classe e de classes derivadas.
- **Quando usar:** Quando subclasses precisam de acesso controlado a membros.

### 4. `internal`
- **Uso:** Acesso permitido somente dentro do mesmo assembly (projeto).

### 5. `protected internal`
- **Uso:** Acesso dentro do mesmo assembly **ou** por herança.

### 6. `private protected` (C# 7.2+)
- **Uso:** Acesso somente por herança **e** dentro do mesmo assembly.

### Resumo visual

```
Modificador          | Mesma classe | Classe derivada (mesmo assembly) | Mesmo assembly | Classe derivada (outro assembly) | Qualquer lugar
---------------------|-------------|-------------------------------|----------------|-------------------------------|---------------
public               | ✅          | ✅                             | ✅              | ✅                             | ✅
private              | ✅          | ❌                             | ❌              | ❌                             | ❌
protected            | ✅          | ✅                             | ❌              | ✅                             | ❌
internal             | ✅          | ✅                             | ✅              | ❌                             | ❌
protected internal   | ✅          | ✅                             | ✅              | ✅                             | ❌
private protected    | ✅          | ✅                             | ❌              | ❌                             | ❌
```

## Modificadores de Classes

### 1. `static`
- **Uso:** Membros ou classes que pertencem ao tipo, não à instância.

### 2. `abstract`
- **Uso:** Define uma assinatura obrigatória em classes base.
- **Quando usar:** Em classes que são apenas "modelos", com lógica deixada para subclasses.

### 3. `virtual` / `override`
- `virtual` diz que um método ou propriedade pode ser sobrescrito por uma classe que herda.
- `override` é usado para sobrescrever.
- **Uso:** Para permitir que métodos sejam sobrescritos.

### 4. `sealed`
- **Uso:** Impede que uma classe ou método seja herdado/sobrescrito.

## Modificadores de Atributos (Campos)

### 1. `readonly`
- **Uso:** Define que um campo só pode ser atribuído no construtor ou na declaração.

### 2. `const`
- **Uso:** Define um valor fixo em tempo de compilação.
- **Quando usar:** Para valores fixos (ex: `PI`, tempo limite).
- **Quando evitar:** Se o valor pode depender de ambiente ou mudar com o tempo.

### 3. `partial`
- **Uso:** Permite dividir a definição de uma classe/método/struct em vários arquivos.

## Interface vs Classe Abstrata

| Característica | Interface | Classe Abstrata |
|---|---|---|
| Herança múltipla | Sim (pode implementar várias) | Não (só pode herdar de uma) |
| Construtores | Não | Sim |
| Campos (state) | Não (só propriedades via contrato) | Sim |
| Implementação padrão | Sim (C# 8+ default methods) | Sim |
| Quando usar | Definir contratos que várias classes não relacionadas implementam | Quando há código compartilhado entre classes relacionadas |

**Regra prática:** você só pode estender uma classe (abstrata ou não), mas pode implementar várias interfaces.

```csharp
// Interface - contrato
public interface INotificavel
{
    void Notificar(string mensagem);
}

// Classe abstrata - comportamento compartilhado
public abstract class EntidadeBase
{
    public Guid Id { get; } = Guid.NewGuid();
    public DateTime CriadoEm { get; } = DateTime.UtcNow;
    
    public abstract void Validar();
}

// Pode herdar de uma classe E implementar múltiplas interfaces
public class Usuario : EntidadeBase, INotificavel
{
    public string Nome { get; set; }
    
    public override void Validar()
    {
        if (string.IsNullOrEmpty(Nome))
            throw new InvalidOperationException("Nome é obrigatório");
    }
    
    public void Notificar(string mensagem)
    {
        Console.WriteLine($"Notificação para {Nome}: {mensagem}");
    }
}
```

## Equals() vs ==

### Operador `==`
- Para **value types** (`int`, `double`, `struct`): compara os **valores**.
- Para **reference types** (`class`): compara as **referências** (se apontam para o mesmo objeto na memória).
- `string` é uma exceção: `==` compara o **conteúdo** (porque o operador é sobrecarregado).

### Método `Equals()`
- Pode ser **sobrescrito** para definir igualdade customizada.
- Por padrão em classes: compara referência (mesma coisa que `==`).
- Por padrão em structs: compara valores campo a campo (mas com reflexão — lento).

```csharp
var a = new Pessoa("João");
var b = new Pessoa("João");

Console.WriteLine(a == b);      // False (referências diferentes)
Console.WriteLine(a.Equals(b)); // False (por padrão, compara referência)

// Após sobrescrever Equals:
public override bool Equals(object obj)
{
    return obj is Pessoa p && p.Nome == Nome;
}

Console.WriteLine(a.Equals(b)); // True (agora compara por valor)
```

**Boa prática:** ao sobrescrever `Equals`, sempre sobrescreva `GetHashCode` também.

---

[← Anterior: Tipos Numéricos](04-tipos-numericos.md) | [Voltar ao índice](README.md)
