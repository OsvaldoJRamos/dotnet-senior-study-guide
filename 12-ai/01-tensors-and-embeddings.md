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
var embedding = await embeddingService.GenerateEmbeddingAsync("texto para converter");
// embedding = float[] with hundreds of dimensions
```

---

[Next: OpenAI API →](02-openai-api.md) | [Back to index](README.md)
