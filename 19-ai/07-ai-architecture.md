# AI Architecture and Real-World Scenarios

## AI-Powered Customer Support System

Core components for a typical AI support system:

```
[User] ←─SignalR─→ [.NET Web API]
                        ↓
                [Semantic Kernel] ← orchestration
                    ↓         ↓
            [Azure OpenAI]  [Plugins]
                              ↓
                    ┌─────────┼──────────┐
                    ↓         ↓          ↓
                [CRM]    [Orders]   [Tickets]
                    ↓
              [Azure AI Search] ← RAG (knowledge base)
```

- System prompt defines behavior, escalation rules, constraints
- Conversation history stored per session in database
- Filters for logging, token tracking, content moderation
- Fallback path to human agents when AI can't resolve

## Integrating AI into a Legacy .NET System

**Don't inject LLM calls into the monolith.** Add AI as a separate service:

1. **Identify high-value use cases** — repetitive tasks humans currently do (classifying tickets, summarizing documents, extracting data)
2. **Create a separate AI service** — new Web API or class library that encapsulates all AI logic
3. **Wrap with your own interface** — `IAIService` so you can mock and swap providers
4. **RAG pipeline** — background job that ingests from legacy DB, chunks, embeds, stores
5. **Human-in-the-loop first** — AI suggests, humans approve. Build trust before automating
6. **Monitor** — accuracy, usage, cost, latency

> Don't scatter `await openAiClient.Complete(...)` throughout the monolith's codebase.

## Handling Sensitive Data

| Strategy | Description |
|----------|-------------|
| **Azure OpenAI** | Data stays in your Azure tenant, not used for training |
| **PII redaction** | Detect and replace names, emails, SSNs with tokens before LLM call |
| **On-premise models** | Llama, Mistral on your own infra for highly sensitive domains |
| **Encryption** | Always encrypt in transit and at rest |
| **Audit logging** | Log prompts and responses but mask sensitive fields |

## Performance Optimization

| Layer | Strategy |
|-------|----------|
| **Model** | Use smaller models for simple tasks (GPT-4o-mini for classification) |
| **Streaming** | Don't wait for full response — stream tokens immediately |
| **Caching** | Cache frequent queries (semantic caching for 40-60% hit rate) |
| **Prompts** | Shorter system prompts = fewer tokens = faster |
| **RAG** | Fewer chunks + re-ranking = less context tokens |
| **Architecture** | Background jobs for non-interactive tasks |
| **Batch API** | 50% cheaper for non-real-time workloads |

## Migrating from Rules to AI

Don't replace entirely on day one — gradual migration:

```
Phase 1: Shadow mode     → AI runs alongside rules, results logged but not used
Phase 2: Edge cases      → AI handles what rules can't (ambiguous cases)
Phase 3: Gradual shift   → Replace rule groups one by one
Phase 4: AI-first        → AI handles everything, rules as safety net
```

> Rules are transparent. AI is a black box. Stakeholders need time to trust it.

## Resilience for AI Services

Treat the LLM as an external dependency — same patterns as any third-party API:

| Pattern | Implementation |
|---------|---------------|
| **Retry** | Exponential backoff with jitter (Polly) on 429 and 5xx |
| **Circuit breaker** | Stop calling after 5 consecutive failures, wait 30s |
| **Fallback model** | Secondary provider (OpenAI direct, self-hosted) |
| **Graceful degradation** | System works without AI — just without AI features |
| **Queue-based** | Non-interactive tasks go through message queue |
| **Timeout** | 30s max — fail fast, show fallback message |

## Testing AI Systems

### Unit tests (no LLM calls)

- Test plugins in isolation — known inputs → expected outputs
- Test prompt construction — user input + chunks → correct prompt
- Test response parsing — known JSON → correct object
- **Mock** the LLM service with deterministic responses

### Integration tests (with LLM calls)

- Golden test set: curated inputs with expected outputs
- Assert on **structure** (valid JSON?), not exact wording
- Use temperature 0 for reproducibility
- Run in CI with a budget limit — they cost real money

### RAG-specific tests

- **Retrieval**: given a question, do the correct chunks come back?
- **Faithfulness**: does the response only contain info from the context?
- **Negative**: given no relevant chunks, does the model say "I don't know"?

### Prompt regression tests

When you change a prompt, run the full test suite. Track metrics across versions: accuracy, latency, token usage, cost.

## Cost Optimization Checklist

1. **Model tiering** — GPT-4o-mini for 80% of requests (60-80% cost reduction)
2. **Semantic caching** — similar questions return cached responses (40-60% hit rate)
3. **Prompt optimization** — compress system prompts, remove redundancy
4. **RAG optimization** — retrieve 20 chunks, re-rank to top 3
5. **Batch API** — 50% cheaper for non-interactive workloads
6. **Token tracking dashboard** — measure cost per feature, per tenant, per day
7. **Output limits** — set `max_tokens` appropriately

## Key Concepts Quick Reference

| Concept | Definition |
|---------|------------|
| **ReAct** | Reasoning + Acting — agent thinks, calls tool, observes result, repeats |
| **AI Agent vs Chatbot** | Chatbot: one question, one answer. Agent: plans multi-step actions, uses tools, loops autonomously |
| **Semantic caching** | Cache by meaning (embeddings), not exact query match |
| **Model fallback** | Cheap model default, expensive model for complex queries |
| **HNSW** | Hierarchical Navigable Small World — algorithm for approximate nearest neighbor search |
| **JSON-RPC** | Lightweight RPC protocol — used by MCP for client-server communication |

---

[← Previous: MCP](06-mcp.md) | [Back to index](README.md)
