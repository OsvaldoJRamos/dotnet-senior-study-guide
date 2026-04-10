# OpenAI / Azure OpenAI API

## OpenAI vs Azure OpenAI

Both offer the same models (GPT-4o, GPT-4-turbo, etc.), but Azure OpenAI runs on Microsoft's infrastructure.

| Aspect | OpenAI Direct | Azure OpenAI |
|--------|---------------|--------------|
| Setup | Simple API key | Azure subscription + deployment |
| Compliance | Basic | SOC2, HIPAA, GDPR |
| Network | Public internet | VNets, private endpoints |
| Auth | API key | Azure AD / RBAC |
| Data residency | US | Regional (choose your region) |
| SLA | Best-effort | Enterprise SLA |
| Best for | Prototyping | Production enterprise |

## Chat Completions API

You send a POST request with an array of messages, each with a role:

- **system** — sets behavior, rules, and constraints
- **user** — the person's input
- **assistant** — previous model responses

```csharp
var client = new ChatClient("gpt-4o", apiKey);

var messages = new List<ChatMessage>
{
    new SystemChatMessage("You are a helpful assistant that answers questions about .NET"),
    new UserChatMessage("What is dependency injection?")
};

var response = await client.CompleteChatAsync(messages);
Console.WriteLine(response.Value.Content[0].Text);
```

> The model is **stateless** — it has no memory between requests. You must send the full conversation history every time.

### Key parameters

| Parameter | Description | Typical values |
|-----------|-------------|----------------|
| `temperature` | Controls randomness (creativity) | 0-0.3 for factual, 0.7-1.0 for creative |
| `max_tokens` | Maximum response length | Depends on use case |
| `top_p` | Nucleus sampling (alternative to temperature) | Usually leave at 1 |

## Streaming

Without streaming, the client waits until the entire response is generated (5-15 seconds for long answers). With streaming, the API returns tokens one by one via **Server-Sent Events (SSE)**.

```csharp
await foreach (var update in client.CompleteChatStreamingAsync(messages))
{
    if (update.ContentUpdate.Count > 0)
    {
        Console.Write(update.ContentUpdate[0].Text); // prints token by token
    }
}
```

If your system uses SignalR, you can forward each token to the frontend in real time — creating the "typing" effect seen in ChatGPT.

## Token Management and Costs

Every model has a **context window** (e.g., GPT-4o supports 128K tokens). Every request uses:
- **Input tokens**: system prompt + history + user message (cheaper)
- **Output tokens**: the response (more expensive)

### Cost optimization strategies

1. **Keep system prompts concise** — every token is sent with every request
2. **Trim conversation history** — summarize old messages instead of sending everything
3. **Set `max_tokens`** — limit response length
4. **Use cheaper models for simple tasks** — GPT-4o-mini instead of GPT-4o
5. **Implement caching** — for repeated/similar queries
6. **Monitor usage** — the API returns `usage.prompt_tokens` and `usage.completion_tokens` in every response

### Model tiering for cost reduction

```
User request → [Classifier] → Simple? → GPT-4o-mini ($0.15/1M input tokens)
                             → Complex? → GPT-4o ($2.50/1M input tokens)
```

This alone can cut costs **60-80%**.

## Rate Limits and Error Handling

The API enforces limits per minute (RPM) and tokens per minute (TPM). When hit, you get a **429 status code** with a `Retry-After` header.

```csharp
// Exponential backoff with jitter using Polly
builder.Services.AddHttpClient("openai")
    .AddTransientHttpErrorPolicy(p =>
        p.WaitAndRetryAsync(3, attempt =>
            TimeSpan.FromSeconds(Math.Pow(2, attempt))
            + TimeSpan.FromMilliseconds(Random.Shared.Next(0, 1000))));
```

Also handle 500/503 errors (service unavailable) with the same retry logic.

## Context Window Overflow

If you exceed the context window, the API returns an error. Prevention:

1. **Count tokens before sending** (using a tokenizer like `tiktoken`)
2. **Truncate older messages** from the conversation history
3. **Summarize the history** — ask the model to summarize, then use the summary
4. **Sliding window** — keep only the last N messages plus the system prompt

## Structured Output

Forces the model to respond in a specific JSON schema — critical for system integrations:

```csharp
var options = new ChatCompletionOptions
{
    ResponseFormat = ChatResponseFormat.CreateJsonSchemaFormat(
        "classification",
        BinaryData.FromObjectAsJson(new
        {
            type = "object",
            properties = new
            {
                category = new { type = "string", @enum = new[] { "billing", "technical", "general" } },
                confidence = new { type = "number" },
                summary = new { type = "string" }
            },
            required = new[] { "category", "confidence", "summary" }
        }))
};
```

> Use structured output whenever the LLM output feeds into another system — APIs, databases, message queues.

---

[← Previous: Tensors and Embeddings](01-tensors-and-embeddings.md) | [Next: Prompt Engineering →](03-prompt-engineering.md) | [Back to index](README.md)
