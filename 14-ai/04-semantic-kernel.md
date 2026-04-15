# Semantic Kernel and Function Calling

## What is Semantic Kernel

Microsoft's open-source SDK for integrating LLMs into .NET applications. Instead of raw API calls, SK provides abstractions:

- **Kernel** — central orchestrator
- **Plugins** — functions exposed to the LLM
- **Filters** — middleware (logging, validation, telemetry)
- **Memory** — semantic search
- **Planners** — multi-step task execution

Integrates natively with Azure OpenAI and follows .NET patterns (DI, strong typing).

## Kernel Setup

```csharp
// Standalone
var kernel = Kernel.CreateBuilder()
    .AddAzureOpenAIChatCompletion("gpt-4o", endpoint, apiKey)
    .Build();

// In ASP.NET Core with DI
builder.Services.AddKernel()
    .AddAzureOpenAIChatCompletion("gpt-4o", endpoint, apiKey);
```

## Plugins

Collections of functions the LLM can discover and call. Two types:

### Native functions (C# methods)

```csharp
public class OrderPlugin
{
    [KernelFunction("get_order_status")]
    [Description("Gets the current status of an order by order number")]
    public async Task<OrderStatus> GetOrderStatus(
        [Description("The order number, e.g. ORD-12345")] string orderNumber)
    {
        return await _orderService.GetStatusAsync(orderNumber);
    }

    [KernelFunction("initiate_return")]
    [Description("Initiates a return process. Requires order number and reason.")]
    public async Task<ReturnResult> InitiateReturn(
        [Description("The order number")] string orderNumber,
        [Description("Reason for the return")] string reason)
    {
        return await _orderService.InitiateReturnAsync(orderNumber, reason);
    }
}

// Register
kernel.Plugins.AddFromType<OrderPlugin>();
// or with DI
kernel.Plugins.AddFromObject(serviceProvider.GetRequiredService<OrderPlugin>());
```

### Prompt functions (template strings)

```csharp
var summarize = kernel.CreateFunctionFromPrompt(
    "Summarize the following text in 3 bullet points: {{$input}}");
```

## Function Calling

Lets the LLM decide **when** to invoke external functions and with **what parameters**.

### Flow

```
1. Define tools (plugins with [KernelFunction])
2. Send user message + tool definitions to the LLM
3. LLM returns tool_calls (function name + arguments as JSON)
4. Your code executes the function
5. Send result back to the LLM
6. LLM generates final response using the result
```

> The model **never executes anything** — it only decides what to call and with what parameters.

> **Terminology note:** OpenAI renamed `functions` → `tools` at DevDay in November 2023. The older `function_call` / `functions` fields are deprecated; the current API uses `tool_calls` / `tools`. In SK, `FunctionChoiceBehavior` (e.g. `Auto()`, `Required()`, `None()`) maps to the new tools API.

### Auto function calling in SK

```csharp
var settings = new OpenAIPromptExecutionSettings
{
    FunctionChoiceBehavior = FunctionChoiceBehavior.Auto() // SK handles the entire loop
};

var result = await kernel.InvokePromptAsync("What is the status of order ORD-456?", 
    new KernelArguments(settings));
```

SK automatically: sends plugins as tools → handles tool calls → invokes plugin methods → sends results back → repeats until the model returns text.

### Parallel function calling

The model can request **multiple function calls** in a single response:

```
User: "Compare the weather in São Paulo and New York"
LLM returns: [get_weather("São Paulo"), get_weather("New York")]  // parallel
```

Execute with `Task.WhenAll` for better performance.

## Conversation History

```csharp
var history = new ChatHistory();
history.AddSystemMessage("You are a helpful assistant.");
history.AddUserMessage("What is my order status?");

var response = await chatService.GetChatMessageContentAsync(history, settings, kernel);
history.AddAssistantMessage(response.Content);

// You manage the history size — truncate or summarize when too large
```

## Filters (Middleware)

Intercept function invocations and prompt rendering:

| Filter Type | When it runs |
|-------------|-------------|
| `IFunctionInvocationFilter` | Before/after any function call |
| `IPromptRenderFilter` | Before a prompt is sent to the model |
| `IAutoFunctionInvocationFilter` | Specifically for auto function calling |

```csharp
public class LoggingFilter : IFunctionInvocationFilter
{
    public async Task OnFunctionInvocationAsync(FunctionInvocationContext context, 
        Func<FunctionInvocationContext, Task> next)
    {
        _logger.LogInformation("Calling {Function}", context.Function.Name);
        var sw = Stopwatch.StartNew();
        
        await next(context);
        
        _logger.LogInformation("Completed in {Elapsed}ms", sw.ElapsedMilliseconds);
    }
}

kernel.FunctionInvocationFilters.Add(new LoggingFilter());
```

## Security Concerns with Function Calling

- **Prompt injection** — malicious input tricks the model into calling functions unintentionally
- **Always validate and sanitize** function arguments before execution
- **Authorization checks** — can this user perform this action?
- **Never execute destructive operations** (DELETE, UPDATE) without confirmation
- **Log all function calls** for auditing
- **Rate limiting** per user

> Treat function calling like any other API endpoint — with proper input validation and access control.

## SK vs LangChain

| Aspect | Semantic Kernel | LangChain |
|--------|----------------|-----------|
| Primary language | .NET (+ Python, Java) | Python (+ JS) |
| Ecosystem | Azure / Microsoft | Python ecosystem |
| DI support | Native | Manual |
| Observability | Azure Monitor, App Insights | LangSmith |
| Agent workflows | Planners, Agent Framework | LangGraph |
| Best for | .NET teams on Azure | Python teams, startups |

---

[← Previous: Prompt Engineering](03-prompt-engineering.md) | [Next: RAG →](05-rag.md) | [Back to index](README.md)
