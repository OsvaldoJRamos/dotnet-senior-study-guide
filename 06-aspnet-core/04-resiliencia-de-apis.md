# Resiliencia de APIs

## O que e

Capacidade de uma API continuar funcionando de forma aceitavel mesmo quando ocorrem **falhas, lentidao, picos de carga ou dependencias instáveis**.

Nao e evitar falhas - e **falhar de forma controlada** e **se recuperar rapido**.

## Por que e importante

Em sistemas distribuidos (microservices, integrações externas, cloud):

- Dependencias externas caem
- Redes falham
- Servicos ficam lentos
- Picos de trafego acontecem
- Deploys causam instabilidade

Sem resiliencia, uma falha pequena vira **efeito domino**.

## Padroes de resiliencia

### 1. Retry (tentativas automaticas)

Repetir a chamada quando falhas **transitorias** ocorrem (timeout, 5xx, conexao resetada).

```csharp
// Com Polly
builder.Services.AddHttpClient("api")
    .AddTransientHttpErrorPolicy(p => 
        p.WaitAndRetryAsync(3, attempt => 
            TimeSpan.FromSeconds(Math.Pow(2, attempt)))); // backoff exponencial
```

> Retry sem controle **aumenta carga** e piora a falha. Sempre use **backoff exponencial**.

### 2. Timeout

Nao esperar para sempre. Cada chamada externa deve ter timeout explicito.

```csharp
builder.Services.AddHttpClient("api")
    .AddPolicyHandler(Policy.TimeoutAsync<HttpResponseMessage>(
        TimeSpan.FromSeconds(10)));
```

Evita threads presas e pool esgotado.

### 3. Circuit Breaker

"Desliga" chamadas para um servico que esta falhando:

- Apos N falhas -> circuito **abre** (bloqueia chamadas)
- Depois de um tempo -> **half-open** (testa com uma chamada)
- Se funcionar -> circuito **fecha** (volta ao normal)

```csharp
builder.Services.AddHttpClient("api")
    .AddTransientHttpErrorPolicy(p =>
        p.CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30)));
```

### 4. Fallback

Resposta alternativa quando algo falha:

- Retornar **cache**
- Retornar **valor padrao**
- **Degradar** funcionalidade
- **Mensagem amigavel**

```csharp
var fallbackPolicy = Policy<HttpResponseMessage>
    .Handle<HttpRequestException>()
    .FallbackAsync(new HttpResponseMessage(HttpStatusCode.OK)
    {
        Content = new StringContent("[]") // retorna lista vazia como fallback
    });
```

### 5. Bulkhead (isolamento)

Isola recursos para que a falha de um nao derrube todos:

```csharp
var bulkhead = Policy.BulkheadAsync<HttpResponseMessage>(
    maxParallelization: 10,    // maximo de chamadas simultaneas
    maxQueuingActions: 20);    // maximo na fila de espera
```

### 6. Rate Limiting

Limita a taxa de requisicoes para proteger o servico:

```csharp
// .NET 7+
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("fixed", opt =>
    {
        opt.Window = TimeSpan.FromSeconds(10);
        opt.PermitLimit = 100;
    });
});
```

## Implementacao tipica com HttpClientFactory + Polly

```csharp
builder.Services.AddHttpClient("pagamento", client =>
{
    client.BaseAddress = new Uri("https://api.pagamento.com");
    client.Timeout = TimeSpan.FromSeconds(30);
})
.AddTransientHttpErrorPolicy(p => p.WaitAndRetryAsync(3, 
    attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt))))
.AddTransientHttpErrorPolicy(p => p.CircuitBreakerAsync(5, 
    TimeSpan.FromSeconds(30)));
```

## Resumo

O objetivo e evitar efeito domino, proteger recursos internos e melhorar a experiencia do usuario mesmo quando algo da errado.

---

[← Anterior: OAuth 2.0](03-oauth2.md) | [Próximo: Middleware →](05-middleware.md) | [Voltar ao índice](README.md)
