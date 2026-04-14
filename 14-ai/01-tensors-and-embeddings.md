# AI - Tensors and Embeddings

## Tensors

A tensor is a generalization of vectors and matrices to N dimensions:

| Dimensions | Name | Example |
|------------|------|---------|
| 0 | Scalar | `42` |
| 1 | Vector | `[1, 2, 3]` |
| 2 | Matrix | `[[1, 2], [3, 4]]` |
| N | Tensor | Matrices of matrices of matrices... |

Tensors are the **fundamental data structure** used in machine learning frameworks (TensorFlow, PyTorch).

## Embeddings

### What they are

Embeddings convert real-world objects (text, images, audio) into **mathematical representations** (vectors) that capture properties and relationships.

Word embedding is a **Natural Language Processing (NLP)** technique that represents words as numbers so that the computer can work with them.

### How it works

Each word/phrase is converted into a **vector of numbers** in a multidimensional space:

```
"king"   -> [0.2, 0.8, 0.1, 0.9, ...]
"queen"  -> [0.2, 0.7, 0.9, 0.8, ...]
"cat"    -> [0.9, 0.1, 0.3, 0.2, ...]
```

Words with similar meanings are **close** in the vector space:

```
"king" - "man" + "woman" ≈ "queen"
```

### Practical use

- **Semantic search**: find documents by meaning, not just keywords
- **Recommendation systems**: find similar items
- **Text classification**: categorize documents
- **Chatbots and LLMs**: understand context and meaning

### In C# with Semantic Kernel

```csharp
// Conceptual example with Semantic Kernel
var kernel = Kernel.CreateBuilder()
    .AddOpenAITextEmbeddingGeneration("text-embedding-3-small", apiKey)
    .Build();

var embeddingService = kernel.GetRequiredService<ITextEmbeddingGenerationService>();
var embedding = await embeddingService.GenerateEmbeddingAsync("text to convert");
// embedding = float[] with hundreds of dimensions
```

### The `dimensions` parameter (OpenAI `text-embedding-3-*`)

OpenAI's `text-embedding-3-small` (default 1536 dims) and `text-embedding-3-large` (default 3072 dims) support a **`dimensions`** parameter that **truncates** the output vector to a smaller size. This uses **Matryoshka Representation Learning** — the model is trained so that the first N dimensions are still a useful embedding on their own.

```csharp
// Request a shorter, cheaper vector — useful to reduce storage, memory, and search latency
var response = await openAiClient.GetEmbeddingsAsync(new EmbeddingsOptions
{
    DeploymentName = "text-embedding-3-large",
    Input = { "text to convert" },
    Dimensions = 512 // truncate from 3072 -> 512
});
```

> Use a smaller `dimensions` to trade a small amount of retrieval quality for big wins in cost and latency (typical sweet spots: 512, 768, 1024). This parameter is **not available** on older `text-embedding-ada-002`.

## Similarity formulas

Once you have two embeddings, you need to measure **how close** they are. The three formulas every senior engineer should recognize:

| Metric | Formula | Range | Notes |
|--------|---------|-------|-------|
| **Cosine similarity** | `(A · B) / (‖A‖ · ‖B‖)` | -1 to 1 (1 = identical direction) | Ignores magnitude. Default for semantic search. |
| **Dot product** | `A · B = Σ aᵢ · bᵢ` | Unbounded | Cheapest to compute. Sensitive to magnitude. |
| **Euclidean distance** | `√Σ (aᵢ - bᵢ)²` | 0 to ∞ (0 = identical) | Actual geometric distance. Lower = more similar. |

> **Important:** OpenAI's `text-embedding-3-*` vectors are **L2-normalized** (unit length). When vectors are normalized, **cosine similarity equals dot product** — so vector databases can use the cheaper dot-product path and get the same ranking.

---

[Next: OpenAI API →](02-openai-api.md) | [Back to index](README.md)
