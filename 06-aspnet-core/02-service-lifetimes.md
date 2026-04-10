# Service Lifetimes: Singleton vs Scoped vs Transient

## Resumo

| Lifetime | Instância por... | Reuso? | Melhor para... | Exemplo |
|---|---|---|---|---|
| **Singleton** | Toda a vida da aplicação | Sempre | Serviços compartilhados e stateless | Logger, Configuration |
| **Scoped** | Request HTTP (web apps) | Por request | Serviços específicos por request | Database context |
| **Transient** | Toda injeção/request | Nunca | Lógica leve e stateless | Email service, Validators |

## Singleton

Classes que serão criadas **uma única vez** durante toda a duração da aplicação. Isso significa menos uso de memória mas também significa que, o que quer que elas façam, devem ser **thread safe**.

O logging é um bom exemplo de singleton porque a implementação interna, mesmo ao escrever em arquivos, garante que os logs são escritos em ordem e nenhuma race condition acontece. Um cache in-memory é outro bom exemplo porque armazena estado que você quer compartilhar por toda a aplicação, mas a implementação deve garantir thread safety.

Outro exemplo é a implementação de um factory pattern. Apesar de poder ser usado como scoped ou transient, como só cria instâncias de uma dada classe, tornar a implementação thread safe e ser singleton vai economizar memória da aplicação.

```csharp
builder.Services.AddSingleton<ILogger, FileLogger>();
builder.Services.AddSingleton<IMemoryCache, MemoryCache>();
```

## Scoped

Classes que são compartilhadas **durante um scope** — geralmente uma web request ou uma ação de usuário em uma aplicação desktop. São perfeitas para implementar o padrão **Unit of Work**, onde você quer compartilhar estado durante a duração daquela operação.

**Conexões com banco de dados e ORM contexts** são bons exemplos porque você quer manter todas as operações de banco dentro da mesma transação, então todos os seus serviços dentro daquele API request ou ação devem compartilhar o mesmo acesso ao banco.

Não pode ser singleton porque operações em paralelo (como diferentes web requests) poderiam afetar o estado umas das outras (você não quer que um rollback de um dado HTTP request reverta outro não relacionado). Mas também não pode ser transient — se ServiceA chama ServiceB, eles receberão conexões diferentes e as mudanças não serão compartilhadas para aquela operação.

```csharp
builder.Services.AddScoped<AppDbContext>();
builder.Services.AddScoped<IUnitOfWork, UnitOfWork>();
```

## Transient

Classes que serão criadas **toda vez** que forem solicitadas. Perfeitas para classes de vida curta e stateless, como serviços que implementam lógica de negócio.

```csharp
builder.Services.AddTransient<IEmailService, SmtpEmailService>();
builder.Services.AddTransient<IValidator<Pedido>, PedidoValidator>();
```

## Regra importante: Captive Dependencies

Nunca injete um serviço de **lifetime menor** em um serviço de **lifetime maior**:

```csharp
// ERRADO - Scoped dentro de Singleton = Captive Dependency
builder.Services.AddSingleton<MeuSingleton>();  // vive pra sempre
builder.Services.AddScoped<MeuScoped>();         // deveria morrer por request

public class MeuSingleton
{
    // MeuScoped vai ficar preso no Singleton e nunca ser descartado!
    public MeuSingleton(MeuScoped scoped) { }
}
```

**Regra:** Singleton > Scoped > Transient (só pode injetar lifetime igual ou maior).

---

[← Anterior: Dependency Injection](01-dependency-injection.md) | [Voltar ao índice](README.md) | [Próximo: OAuth 2.0 →](03-oauth2.md)
