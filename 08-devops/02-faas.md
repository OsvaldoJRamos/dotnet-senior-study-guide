# Function as a Service (FaaS)

## O que e

FaaS (AWS Lambda, Azure Functions, Google Cloud Functions) sao basicamente **containers pequenos** que executam funcoes isoladas.

## Como funciona

1. Voce configura um **evento/trigger** (chamada HTTP, mensagem de fila, pub/sub)
2. Quando o evento chega, um **container sobe** e executa a funcao
3. Apos processar, o container pode ser **desligado** para economizar recursos

```
Evento (HTTP, SQS, etc.) --> Container sobe --> Funcao executa --> Container pode desligar
```

## Caracteristicas

- Containers pre-configurados com runtimes (Node.js, Python, Go, C#, Java, etc.)
- Permite instalar dependencias externas
- Container tende a ser **pequeno** e ficar de pe por periodos **curtos** (horas, no maximo dias)
- A plataforma mantém o container ativo por mais algum tempo caso outros eventos estejam esperando

## Para que e indicado

- **Processamento de eventos**: mensagens de fila, webhooks
- **Tarefas agendadas**: cron jobs leves
- **APIs leves**: endpoints simples e independentes
- **Processamento de arquivos**: resize de imagens, ETL

## Para que NAO e indicado

- **Frameworks web monoliticos** (ASP.NET MVC, Rails, etc.) - tecnicamente possivel, mas nao e o proposito
- **Aplicacoes com estado**: FaaS e stateless por natureza
- **Processamento longo**: tem limite de tempo (ex: Lambda = 15min max)
- **Cold start**: primeira execucao pode ser lenta (container subindo)

## Exemplo com Azure Functions (C#)

```csharp
public class ProcessarPedidoFunction
{
    [FunctionName("ProcessarPedido")]
    public async Task Run(
        [QueueTrigger("pedidos-queue")] string pedidoJson,
        ILogger log)
    {
        var pedido = JsonSerializer.Deserialize<Pedido>(pedidoJson);
        log.LogInformation($"Processando pedido {pedido.Id}");
        
        // processa o pedido...
    }
}
```

## Cold Start

Quando um container e criado do zero, ha um delay (**cold start**). Estrategias para mitigar:

- **Provisioned concurrency** (AWS) / **Always Ready** (Azure) - manter containers quentes
- Usar runtimes leves (Node.js, Go) vs pesados (Java, .NET)
- **AOT compilation** (.NET 8+) - reduz drasticamente o cold start

## Cuidado com vendor lock-in

A implementacao varia entre fornecedores (AWS, Azure, GCP). O framework e os triggers sao especificos da plataforma.

---

[← Anterior: CI/CD](01-ci-cd.md) | [Próximo: Docker e Kubernetes →](03-docker-e-kubernetes.md) | [Voltar ao índice](README.md)
