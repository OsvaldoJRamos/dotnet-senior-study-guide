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

| Role | Responsibility |
|------|----------------|
| **Host** | Application that owns the LLM and the user interaction (Claude Desktop, an IDE, your web app). |
| **Client** | Component **inside the host** that speaks MCP (JSON-RPC 2.0) to servers: discovers tools/resources/prompts and invokes them. |
| **Server** | Lightweight process that exposes tools, resources, and prompts from an external system (DB, API, filesystem). |

> One-liner for interviews: the **server** *exposes* capabilities, the **client** *discovers and invokes* them, the **host** *manages the LLM and user*.

## What an MCP Server Exposes

| Capability | Description | Example |
|-----------|-------------|---------|
| **Tools** | Functions the LLM can call | `query_database`, `send_slack_message` |
| **Resources** | Data the LLM can read | Files, database records, API responses |
| **Prompts** | Reusable prompt templates | Best practices for the server's domain |

Tools are the most commonly used capability.

## Transport Protocols

| Transport | How it works | Status / Best for |
|-----------|-------------|-------------------|
| **Stdio** | Host launches server as child process, communicates via stdin/stdout | Stable — local dev, desktop apps |
| **Streamable HTTP** | HTTP with optional streaming, single endpoint, supports resumability | **Current standard** (spec 2025-03-26+) for remote/production |
| **SSE over HTTP** | Separate SSE + POST endpoints | **Legacy** — replaced by Streamable HTTP; kept for backward compatibility |

## Building an MCP Server in C#

The official .NET SDK is the **`ModelContextProtocol`** NuGet package (preview). Tools are static methods decorated with `[McpServerTool]` inside a class marked `[McpServerToolType]`:

```csharp
using ModelContextProtocol.Server;
using System.ComponentModel;
using System.Text.Json;

[McpServerToolType]
public static class OrderTools
{
    [McpServerTool, Description("Gets the current status of an order")]
    public static async Task<string> GetOrderStatus(
        IOrderService orderService,
        [Description("Order number, e.g. ORD-12345")] string orderNumber)
    {
        var order = await orderService.GetAsync(orderNumber);
        return JsonSerializer.Serialize(new
        {
            order.Number,
            order.Status,
            order.EstimatedDelivery
        });
    }

    [McpServerTool, Description("Lists recent orders for a customer")]
    public static async Task<string> ListOrders(
        IOrderService orderService,
        [Description("Customer email")] string email,
        [Description("Number of orders to return")] int limit = 10)
    {
        var orders = await orderService.ListByCustomerAsync(email, limit);
        return JsonSerializer.Serialize(orders);
    }
}

// Program.cs — host it over stdio
var builder = Host.CreateApplicationBuilder(args);
builder.Services.AddSingleton<IOrderService, OrderService>();
builder.Services
    .AddMcpServer()
    .WithStdioServerTransport()
    .WithToolsFromAssembly();
await builder.Build().RunAsync();
```

## MCP + Semantic Kernel Integration

Connect an MCP client to a server, then expose the server's tools to Semantic Kernel as kernel functions:

```csharp
using ModelContextProtocol.Client;

// 1. Create a transport pointing at the MCP server process (stdio here)
var transport = new StdioClientTransport(new StdioClientTransportOptions
{
    Command = "dotnet",
    Arguments = ["run", "--project", "OrderMcpServer"]
});

// 2. Create the client over that transport
var client = await McpClientFactory.CreateAsync(transport);

// 3. List the server's tools and register them as SK functions
var tools = await client.ListToolsAsync();
kernel.Plugins.AddFromFunctions("Mcp", tools.Select(t => t.AsKernelFunction()));

// 4. Auto function calling works seamlessly with the rest of SK
var result = await kernel.InvokePromptAsync(
    "What is the status of order ORD-456?",
    new KernelArguments(new OpenAIPromptExecutionSettings
    {
        FunctionChoiceBehavior = FunctionChoiceBehavior.Auto()
    }));
```

> For remote servers, swap `StdioClientTransport` for `SseClientTransport` (or the newer Streamable HTTP transport) pointed at the server URL.

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
