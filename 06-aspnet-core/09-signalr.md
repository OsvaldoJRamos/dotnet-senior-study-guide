# SignalR (Real-Time Communication)

## What it is

SignalR is an ASP.NET Core library for **bidirectional real-time communication** between server and client. It abstracts WebSockets, Server-Sent Events, and Long Polling.

## When to use

- Real-time **chat**
- **Dashboards** with live updates
- Push **notifications**
- **Collaboration** (like Google Docs)
- Multiplayer **gaming**

## How it works

```
Client  ←──WebSocket──→  Hub (server)  ←──→  Other clients
```

1. Client connects to the **Hub**
2. Hub can send messages to:
   - A **specific** client
   - **All** clients
   - A **group** of clients

## Implementation

### Server (Hub)

```csharp
public class ChatHub : Hub
{
    // Client calls this method
    public async Task EnviarMensagem(string usuario, string mensagem)
    {
        // Sends to ALL connected clients
        await Clients.All.SendAsync("ReceberMensagem", usuario, mensagem);
    }

    // Send to a specific group
    public async Task EnviarParaSala(string sala, string mensagem)
    {
        await Clients.Group(sala).SendAsync("ReceberMensagem", mensagem);
    }

    // Join a room
    public async Task EntrarNaSala(string sala)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, sala);
    }

    // Connection events
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

// Receive messages
connection.on("ReceberMensagem", (usuario, mensagem) => {
    console.log(`${usuario}: ${mensagem}`);
});

// Send message
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

## Sending from outside the Hub

Useful in controllers or background services:

```csharp
public class PedidoController : ControllerBase
{
    private readonly IHubContext<NotificacaoHub> _hub;

    [HttpPost]
    public async Task<IActionResult> Criar(PedidoDto dto)
    {
        var pedido = await _service.CriarAsync(dto);

        // Notify connected clients
        await _hub.Clients.All.SendAsync("PedidoCriado", pedido.Id);

        return CreatedAtAction(nameof(Obter), new { id = pedido.Id }, pedido);
    }
}
```

## Scaling with Redis Backplane

With multiple instances, clients on different instances can't see each other. Solution: **Redis backplane**.

```csharp
builder.Services.AddSignalR()
    .AddStackExchangeRedis("localhost:6379", options =>
    {
        options.Configuration.ChannelPrefix = RedisChannel.Literal("minha-app");
    });
```

## Transports

| Transport | Description |
|-----------|-----------|
| **WebSockets** | Bidirectional, most efficient (preferred) |
| **Server-Sent Events** | Server to client only |
| **Long Polling** | Fallback compatible with everything |

SignalR automatically negotiates the best available transport.

---

[← Previous: Minimal APIs](08-minimal-apis.md) | [Back to index](README.md)
