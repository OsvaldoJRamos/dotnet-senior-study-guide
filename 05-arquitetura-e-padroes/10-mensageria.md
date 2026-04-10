# Mensageria (Message Brokers)

## O que e

Mensageria e o padrao de comunicacao **assincrona** entre servicos usando um intermediario (broker). O produtor envia a mensagem e **nao espera** a resposta.

```
[Produtor] ──publica──→ [Broker] ──entrega──→ [Consumidor]
```

## Por que usar

- **Desacoplamento**: produtor nao precisa saber quem consome
- **Resiliencia**: se o consumidor estiver fora, a mensagem fica na fila
- **Escala**: multiplos consumidores processam em paralelo
- **Picos de carga**: fila absorve o pico, consumidores processam no ritmo deles

## Padroes de mensageria

### 1. Queue (ponto a ponto)

Uma mensagem e processada por **um unico consumidor**:

```
[Producer] → [Queue] → [Consumer 1]
                      → [Consumer 2]  (so um recebe cada mensagem)
```

Uso: processamento de pedidos, envio de emails, jobs assincronos.

### 2. Pub/Sub (publish/subscribe)

Uma mensagem e entregue a **todos os assinantes**:

```
[Publisher] → [Topic] → [Subscriber 1] (recebe)
                      → [Subscriber 2] (recebe)
                      → [Subscriber 3] (recebe)
```

Uso: notificacoes, sincronizacao entre servicos, event-driven architecture.

## RabbitMQ

Broker de mensagens mais popular para aplicacoes .NET. Baseado no protocolo **AMQP**.

### Conceitos

- **Exchange**: recebe mensagens do produtor e roteia para filas
- **Queue**: armazena mensagens ate serem consumidas
- **Binding**: regra que conecta exchange a queue
- **Routing Key**: chave usada para roteamento

```
[Producer] → [Exchange] ──routing key──→ [Queue] → [Consumer]
```

### Tipos de Exchange

| Tipo | Roteamento |
|------|------------|
| **Direct** | Routing key exata |
| **Fanout** | Todas as filas (broadcast) |
| **Topic** | Routing key com wildcards (`pedido.*`, `#.erro`) |
| **Headers** | Headers da mensagem |

### Exemplo com MassTransit (.NET)

```csharp
// Configuracao
builder.Services.AddMassTransit(x =>
{
    x.AddConsumer<PedidoCriadoConsumer>();

    x.UsingRabbitMq((context, cfg) =>
    {
        cfg.Host("rabbitmq://localhost");
        cfg.ConfigureEndpoints(context);
    });
});

// Publicar evento
public class PedidoService
{
    private readonly IPublishEndpoint _publish;

    public async Task CriarPedido(Pedido pedido)
    {
        // ... salva pedido ...
        await _publish.Publish(new PedidoCriadoEvent(pedido.Id, pedido.Total));
    }
}

// Consumir evento
public class PedidoCriadoConsumer : IConsumer<PedidoCriadoEvent>
{
    public async Task Consume(ConsumeContext<PedidoCriadoEvent> context)
    {
        var evento = context.Message;
        // ... envia email, atualiza estoque, etc.
    }
}
```

## Apache Kafka

Plataforma de **streaming de eventos**. Diferente de filas tradicionais:

- Mensagens sao **persistidas** (log distribuido)
- Consumidores controlam seu **offset** (podem reler mensagens)
- Altissimo throughput (milhoes de mensagens/segundo)

### Conceitos

- **Topic**: categoria de mensagens
- **Partition**: subdivisao de um topic (paralelismo)
- **Consumer Group**: grupo de consumidores que dividem as partitions
- **Offset**: posicao do consumidor no log

### Kafka vs RabbitMQ

| Aspecto | RabbitMQ | Kafka |
|---------|----------|-------|
| Modelo | Message broker (filas) | Event streaming (log) |
| Mensagem consumida | Removida da fila | Permanece no log |
| Reler mensagens | Nao | Sim (por offset) |
| Throughput | Medio-alto | Muito alto |
| Latencia | Mais baixa | Mais alta |
| Uso ideal | Tarefas assincronas, RPC | Event sourcing, analytics, alta escala |
| Complexidade | Menor | Maior |

## Azure Service Bus

Servico gerenciado da Microsoft. Suporta **queues** e **topics/subscriptions**:

```csharp
// Enviar mensagem
var client = new ServiceBusClient(connectionString);
var sender = client.CreateSender("pedidos-queue");

await sender.SendMessageAsync(new ServiceBusMessage(
    JsonSerializer.Serialize(pedido)));

// Receber mensagem
var processor = client.CreateProcessor("pedidos-queue");
processor.ProcessMessageAsync += async args =>
{
    var pedido = JsonSerializer.Deserialize<Pedido>(args.Message.Body.ToString());
    await args.CompleteMessageAsync(args.Message);
};
await processor.StartProcessingAsync();
```

## Garantias de entrega

| Garantia | Descricao | Quando usar |
|----------|-----------|-------------|
| **At-most-once** | Pode perder, nunca duplica | Logs, metricas |
| **At-least-once** | Nunca perde, pode duplicar | Maioria dos casos (com idempotencia) |
| **Exactly-once** | Nunca perde, nunca duplica | Transacoes financeiras (caro/complexo) |

> Na pratica, use **at-least-once** com **consumidores idempotentes**.

---

[← Anterior: Microservices](09-microservices.md) | [Voltar ao índice](README.md)
