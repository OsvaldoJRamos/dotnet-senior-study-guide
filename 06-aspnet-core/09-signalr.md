# SignalR (Comunicacao em Tempo Real)

## O que e

SignalR e uma biblioteca do ASP.NET Core para **comunicacao bidirecional em tempo real** entre servidor e cliente. Abstrai WebSockets, Server-Sent Events e Long Polling.

## Quando usar

- **Chat** em tempo real
- **Dashboards** com atualizacao ao vivo
- **Notificacoes** push
- **Colaboracao** (tipo Google Docs)
- **Gaming** multiplayer

## Como funciona

```
Cliente  ←──WebSocket──→  Hub (servidor)  ←──→  Outros clientes
```

1. Cliente se conecta ao **Hub**
2. Hub pode enviar mensagens para:
   - **Um cliente** especifico
   - **Todos** os clientes
   - **Um grupo** de clientes

## Implementacao

### Server (Hub)

```csharp
public class ChatHub : Hub
{
    // Cliente chama este metodo
    public async Task EnviarMensagem(string usuario, string mensagem)
    {
        // Envia para TODOS os clientes conectados
        await Clients.All.SendAsync("ReceberMensagem", usuario, mensagem);
    }

    // Enviar para grupo especifico
    public async Task EnviarParaSala(string sala, string mensagem)
    {
        await Clients.Group(sala).SendAsync("ReceberMensagem", mensagem);
    }

    // Entrar em uma sala
    public async Task EntrarNaSala(string sala)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, sala);
    }

    // Eventos de conexao
    public override async Task OnConnectedAsync()
    {
        await base.OnConnectedAsync();
        // usuario conectou
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        await base.OnDisconnectedAsync(exception);
        // usuario desconectou
    }
}

// Program.cs
builder.Services.AddSignalR();
app.MapHub<ChatHub>("/chat");
```

### Client (JavaScript)

```javascript
const connection = new signalR.HubConnectionBuilder()
    .withUrl("/chat")
    .withAutomaticReconnect()
    .build();

// Receber mensagens
connection.on("ReceberMensagem", (usuario, mensagem) => {
    console.log(`${usuario}: ${mensagem}`);
});

// Enviar mensagem
await connection.invoke("EnviarMensagem", "Osvaldo", "Olá!");

await connection.start();
```

### Client (C#)

```csharp
var connection = new HubConnectionBuilder()
    .WithUrl("https://localhost:5001/chat")
    .WithAutomaticReconnect()
    .Build();

connection.On<string, string>("ReceberMensagem", (usuario, msg) =>
{
    Console.WriteLine($"{usuario}: {msg}");
});

await connection.StartAsync();
await connection.InvokeAsync("EnviarMensagem", "Bot", "Conectado!");
```

## Enviar de fora do Hub

Util em controllers ou background services:

```csharp
public class PedidoController : ControllerBase
{
    private readonly IHubContext<NotificacaoHub> _hub;

    [HttpPost]
    public async Task<IActionResult> Criar(PedidoDto dto)
    {
        var pedido = await _service.CriarAsync(dto);

        // Notifica clientes conectados
        await _hub.Clients.All.SendAsync("PedidoCriado", pedido.Id);

        return CreatedAtAction(nameof(Obter), new { id = pedido.Id }, pedido);
    }
}
```

## Escala com Redis Backplane

Com multiplas instancias, clientes em instancias diferentes nao se enxergam. Solucao: **Redis backplane**.

```csharp
builder.Services.AddSignalR()
    .AddStackExchangeRedis("localhost:6379", options =>
    {
        options.Configuration.ChannelPrefix = RedisChannel.Literal("minha-app");
    });
```

## Transportes

| Transporte | Descricao |
|-----------|-----------|
| **WebSockets** | Bidirecional, mais eficiente (preferido) |
| **Server-Sent Events** | Servidor → cliente apenas |
| **Long Polling** | Fallback compativel com tudo |

SignalR negocia automaticamente o melhor transporte disponivel.

---

[← Anterior: Minimal APIs](08-minimal-apis.md) | [Voltar ao índice](README.md)
