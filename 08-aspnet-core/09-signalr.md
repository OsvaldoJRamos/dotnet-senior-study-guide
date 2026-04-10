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
    public async Task SendMessage(string user, string message)
    {
        // Sends to ALL connected clients
        await Clients.All.SendAsync("ReceiveMessage", user, message);
    }

    // Send to a specific group
    public async Task SendToRoom(string room, string message)
    {
        await Clients.Group(room).SendAsync("ReceiveMessage", message);
    }

    // Join a room
    public async Task JoinRoom(string room)
    {
        await Groups.AddToGroupAsync(Context.ConnectionId, room);
    }

    // Connection events
    public override async Task OnConnectedAsync()
    {
        await base.OnConnectedAsync();
        // user connected
    }

    public override async Task OnDisconnectedAsync(Exception? exception)
    {
        await base.OnDisconnectedAsync(exception);
        // user disconnected
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
connection.on("ReceiveMessage", (user, message) => {
    console.log(`${user}: ${message}`);
});

// Send message
await connection.invoke("SendMessage", "Osvaldo", "Hello!");

await connection.start();
```

### Client (C#)

```csharp
var connection = new HubConnectionBuilder()
    .WithUrl("https://localhost:5001/chat")
    .WithAutomaticReconnect()
    .Build();

connection.On<string, string>("ReceiveMessage", (user, msg) =>
{
    Console.WriteLine($"{user}: {msg}");
});

await connection.StartAsync();
await connection.InvokeAsync("SendMessage", "Bot", "Connected!");
```

## Sending from outside the Hub

Useful in controllers or background services:

```csharp
public class OrderController : ControllerBase
{
    private readonly IHubContext<NotificationHub> _hub;

    [HttpPost]
    public async Task<IActionResult> Create(OrderDto dto)
    {
        var order = await _service.CreateAsync(dto);

        // Notify connected clients
        await _hub.Clients.All.SendAsync("OrderCreated", order.Id);

        return CreatedAtAction(nameof(Get), new { id = order.Id }, order);
    }
}
```

## Scaling with Redis Backplane

With multiple instances, clients on different instances can't see each other. Solution: **Redis backplane**.

```csharp
builder.Services.AddSignalR()
    .AddStackExchangeRedis("localhost:6379", options =>
    {
        options.Configuration.ChannelPrefix = RedisChannel.Literal("my-app");
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
