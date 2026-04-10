# IA - Tensors e Embeddings

## Tensors

Um tensor e uma generalizacao de vetores e matrizes para N dimensoes:

| Dimensoes | Nome | Exemplo |
|-----------|------|---------|
| 0 | Escalar | `42` |
| 1 | Vetor | `[1, 2, 3]` |
| 2 | Matriz | `[[1, 2], [3, 4]]` |
| N | Tensor | Matrizes de matrizes de matrizes... |

Tensors sao a **estrutura de dados fundamental** usada em frameworks de machine learning (TensorFlow, PyTorch).

## Embeddings

### O que sao

Embeddings convertem objetos do mundo real (texto, imagens, audio) em **representacoes matematicas** (vetores) que capturam propriedades e relacoes.

Embedding de palavras e uma tecnica de **Processamento de Linguagem Natural (PLN)** que representa palavras como numeros para que o computador possa trabalhar com elas.

### Como funciona

Cada palavra/frase e convertida em um **vetor de numeros** em um espaco multidimensional:

```
"rei"    -> [0.2, 0.8, 0.1, 0.9, ...]
"rainha" -> [0.2, 0.7, 0.9, 0.8, ...]
"gato"   -> [0.9, 0.1, 0.3, 0.2, ...]
```

Palavras com significados similares ficam **proximas** no espaco vetorial:

```
"rei" - "homem" + "mulher" ≈ "rainha"
```

### Uso pratico

- **Busca semantica**: encontrar documentos por significado, nao apenas palavras-chave
- **Sistemas de recomendacao**: encontrar itens similares
- **Classificacao de texto**: categorizar documentos
- **Chatbots e LLMs**: entender contexto e significado

### Em C# com Semantic Kernel

```csharp
// Exemplo conceitual com Semantic Kernel
var kernel = Kernel.CreateBuilder()
    .AddOpenAITextEmbeddingGeneration("text-embedding-3-small", apiKey)
    .Build();

var embeddingService = kernel.GetRequiredService<ITextEmbeddingGenerationService>();
var embedding = await embeddingService.GenerateEmbeddingAsync("texto para converter");
// embedding = float[] com centenas de dimensoes
```

---

[Voltar ao índice](README.md)
