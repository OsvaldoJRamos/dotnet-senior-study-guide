# MCP (Model Context Protocol)

## What it is

Open protocol created by Anthropic that standardizes how LLMs connect to external tools and data sources. Often compared to **"USB-C for LLMs"** — a universal connector.

Before MCP, every framework had its own plugin system. MCP provides a universal standard: any MCP server can be used by any MCP client, regardless of the framework.

## Architecture

```
┌─────────────────────────────────────────────────┐
│  MCP Host (your app, Claude Desktop, IDE)        │
│  ┌───────────────────────────────────────────┐  │
│  │  MCP Client (communicates via JSON-RPC)    │  │
│  └────────┬──────────────┬───────────────────┘  │
└───────────┼──────────────┼──────────────────────┘
            ↓              ↓
    ┌───────────────┐ ┌───────────────┐
    │  MCP Server   │ │  MCP Server   │
    │  (Database)   │ │  (Slack)      │
    └───────────────┘ └───────────────┘
```

- **Host** — application running the LLM
- **Client** — component that communicates with servers using JSON-RPC 2.0
- **Server** — lightweight process that wraps an external system

## What an MCP Server Exposes

| Capability | Description | Example |
|-----------|-------------|---------|
| **Tools** | Functions the LLM can call | `query_database`, `send_slack_message` |
| **Resources** | Data the LLM can read | Files, database records, API responses |
| **Prompts** | Reusable prompt templates | Best practices for the server's domain |

Tools are the most commonly used capability.

## Transport Protocols

| Transport | How it works | Best for |
|-----------|-------------|----------|
| **Stdio** | Host launches server as child process, communicates via stdin/stdout | Local dev, desktop apps |
| **SSE over HTTP** | Server runs as HTTP service, client connects via HTTP + SSE | Remote/production, multiple clients |
| **Streamable HTTP** | Evolution of SSE, better bidirectional communication | Emerging standard |

## Building an MCP Server in C#

```csharp
using ModelContextProtocol;

public class OrderMcpServer
{
    [McpServerTool]
    [Description("Gets the current status of an order")]
    public async Task<string> GetOrderStatus(
        [Description("Order number, e.g. PED-12345")] string orderNumber)
    {
        var order = await _orderService.GetAsync(orderNumber);
        return JsonSerializer.Serialize(new
        {
            order.Number,
            order.Status,
            order.EstimatedDelivery
        });
    }

    [McpServerTool]
    [Description("Lists recent orders for a customer")]
    public async Task<string> ListOrders(
        [Description("Customer email")] string email,
        [Description("Number of orders to return")] int limit = 10)
    {
        var orders = await _orderService.ListByCustomerAsync(email, limit);
        return JsonSerializer.Serialize(orders);
    }
}
```

## MCP + Semantic Kernel Integration

SK has native MCP support — import MCP server tools as SK plugins:

```csharp
// Connect to MCP server
var mcpClient = await McpClientFactory.CreateAsync(new McpServerConfig
{
    Command = "dotnet",
    Arguments = ["run", "--project", "OrderMcpServer"]
});

// Import as SK plugins — works like any other plugin
kernel.Plugins.AddFromMcpClient(mcpClient);

// Auto function calling works seamlessly
var result = await kernel.InvokePromptAsync("What is the status of order PED-456?",
    new KernelArguments(new OpenAIPromptExecutionSettings
    {
        FunctionChoiceBehavior = FunctionChoiceBehavior.Auto()
    }));
```

## Function Calling vs MCP

| | Function Calling | MCP |
|-|-----------------|-----|
| **What** | Mechanism — how the LLM invokes functions | Protocol — standardizes tool discovery and invocation |
| **Scope** | API-level, framework-specific | Universal, cross-framework |
| **Analogy** | The engine | The highway system |

MCP **uses** function calling under the hood. When an MCP client sends tools to the LLM, they become function calling tools.

## Advantages over Custom Integrations

- **Reusability** — build a server once, use it from any MCP-compatible host
- **Discoverability** — clients can list available tools automatically
- **Ecosystem** — hundreds of pre-built servers for popular services
- **Composability** — connect multiple servers to one host simultaneously
- **Maintenance** — updating tool logic only requires updating the server

---

[← Previous: RAG](05-rag.md) | [Next: AI Architecture Scenarios →](07-ai-architecture.md) | [Back to index](README.md)
