# RAG (Retrieval-Augmented Generation)

## What it is

RAG solves the problem of LLMs **not knowing your private data**. Instead of fine-tuning the model, you **retrieve relevant documents at query time** and inject them into the prompt as context.

```
User question → [Embed query] → [Search vector DB] → [Retrieve top K chunks]
                                                              ↓
                              [LLM generates answer] ← [Inject chunks as context]
```

### RAG vs Fine-tuning

| Aspect | RAG | Fine-tuning |
|--------|-----|-------------|
| Data updates | Instant (re-index) | Requires retraining |
| Cost | Lower | Higher |
| Citations | Can cite sources | Cannot |
| Model agnostic | Yes | Tied to one model |
| Best for | Frequently changing data, traceability | Specific style/tone, specialized tasks |

> Most enterprise applications use RAG because data changes constantly and traceability matters.

## Embeddings (Deep Dive)

An embedding is a **dense vector** representation of text — typically 1536 or 3072 floating-point numbers. Semantically similar texts produce **similar vectors**.

```
"How do I return a product?"  →  [0.12, 0.87, -0.34, ...]
"What is your refund policy?" →  [0.11, 0.85, -0.31, ...]  ← similar!
"The weather is nice today"   →  [0.92, -0.15, 0.67, ...]  ← very different
```

Similarity is measured with **cosine similarity** — a value between -1 and 1, where 1 means identical meaning.

Models: `text-embedding-3-small` (cheaper, 1536 dims), `text-embedding-3-large` (better, 3072 dims).

## Chunking Strategies

Breaking documents into smaller pieces for embedding and retrieval. The challenge: too large = irrelevant context wasting tokens; too small = lost context.

| Strategy | Description | Best for |
|----------|-------------|----------|
| **Fixed-size** | 500-1000 tokens with 100-200 overlap | Simple, general purpose |
| **Recursive** | Try paragraphs → sentences → characters | Unstructured text |
| **Semantic** | Use model to detect topic boundaries | Mixed-topic documents |
| **Document-aware** | Respect headings, sections, tables | Structured docs (policies, manuals) |

> Overlap between chunks ensures information at boundaries isn't lost.

## Vector Databases

Store embeddings and enable fast similarity search using algorithms like HNSW or IVF.

| Database | Type | Best for |
|----------|------|----------|
| **Azure AI Search** | Managed | .NET/Azure stack, hybrid search |
| **Qdrant** | Self-hosted | Open-source, easy Docker setup |
| **pgvector** | PostgreSQL extension | Already using PostgreSQL |
| **Pinecone** | Managed | Startups, quick setup |
| **ChromaDB** | In-memory | Prototyping |

## Hybrid Search

Pure vector search finds semantically similar content but **can miss exact matches** (like order numbers, product codes). Keyword search (BM25) finds exact terms but **misses synonyms**.

**Hybrid search combines both** using Reciprocal Rank Fusion (RRF):

```
User: "What's the status of PED-98765?"

Vector search → chunks about orders in general (semantic match)
Keyword search → chunk mentioning PED-98765 specifically (exact match)
Hybrid → both, ranked by combined relevance
```

Azure AI Search supports hybrid search natively.

## RAG Pipeline Architecture

```
[Documents] → [Chunk] → [Embed] → [Store in Vector DB]
                                         ↑ (ingestion - offline)
                                         ↓ (query - real-time)
[User query] → [Embed] → [Search] → [Top K chunks] → [LLM + context] → [Answer]
```

### Implementation with Semantic Kernel

```csharp
// Ingestion (background job)
var chunks = ChunkDocument(document, chunkSize: 500, overlap: 100);
foreach (var chunk in chunks)
{
    var embedding = await embeddingService.GenerateEmbeddingAsync(chunk.Text);
    await vectorStore.UpsertAsync(new MemoryRecord
    {
        Id = chunk.Id,
        Embedding = embedding,
        Text = chunk.Text,
        Metadata = new { Source = document.Name, TenantId = tenantId }
    });
}

// Query
var queryEmbedding = await embeddingService.GenerateEmbeddingAsync(userQuestion);
var results = await vectorStore.SearchAsync(queryEmbedding, topK: 5);

var context = string.Join("\n---\n", results.Select(r => r.Text));
var prompt = $"""
    Answer based ONLY on the following context. If the answer is not in the context, say "I don't know."
    
    Context:
    {context}
    
    Question: {userQuestion}
    """;
```

## Evaluating RAG Quality

| Dimension | Metric | Description |
|-----------|--------|-------------|
| **Retrieval** | Precision@K | How many of the top K results are relevant? |
| **Retrieval** | Recall | Did we find all relevant chunks? |
| **Generation** | Faithfulness | Is the answer faithful to the context? (no hallucinations) |
| **Generation** | Completeness | Does the response cover everything relevant? |

Build a **golden test set** of questions with expected answers and source chunks, then run regular evaluations.

## Multi-Tenant RAG

For SaaS products where each tenant has their own knowledge base:

- **Separate indexes per tenant** — most secure, more operational overhead
- **Shared index with tenant filter** — simpler, requires rigorous filter enforcement
- **Tenant ID must come from authenticated session**, never from user input
- Filter enforcement is **server-side**, never client-side

---

[← Previous: Semantic Kernel](04-semantic-kernel.md) | [Next: MCP →](06-mcp.md) | [Back to index](README.md)
