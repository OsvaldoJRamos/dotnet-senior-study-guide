# Prompt Engineering

## System Prompt

The first message with `role: "system"`. Defines the model's persona, behavior rules, output format, language, tone, and constraints.

A well-crafted system prompt is the difference between a generic chatbot and a specialized tool.

```
You are a customer support agent for XYZ Store.
Rules:
- Only answer questions about our products and policies.
- Always respond in JSON format.
- Never make up information — if unsure, say "I don't know".
- Be empathetic but concise.
- If the customer is angry, escalate to a human agent.
```

> The system prompt is sent with every request and consumes tokens — keep it concise but comprehensive.

## Few-Shot Prompting

Providing examples of desired input-output pairs before the actual request. The model learns the pattern and applies it.

```
Classify the following support ticket:

Example 1:
Input: "My invoice shows the wrong amount"
Output: {"category": "billing", "priority": "medium"}

Example 2:
Input: "The server is returning 500 errors"
Output: {"category": "technical", "priority": "high"}

Now classify:
Input: "I can't log into my account"
Output:
```

### When to use few-shot

- Classification tasks (sentiment, ticket categorization)
- Data extraction (parsing addresses, extracting entities)
- Format standardization (converting dates, normalizing names)
- Any task where **showing is easier than explaining**

Typically **2-5 examples** are enough. Zero-shot = no examples, just instructions.

## Chain-of-Thought (CoT)

Instructs the model to **reason step by step** before giving a final answer. Dramatically improves accuracy on complex tasks.

```
Think step by step:
1. Identify what type of issue this is
2. Check if it falls within our support scope
3. Determine the priority based on business impact
4. Provide your classification with reasoning
```

Trigger phrases: "Think step by step", "Let's break this down", or by structuring the prompt with numbered steps.

## Preventing Hallucinations

Hallucination = the model generates plausible but **false** information.

| Strategy | Description |
|----------|-------------|
| **RAG** | Ground responses in real data |
| **Explicit instructions** | "Only answer based on the provided context — if the answer isn't there, say you don't know" |
| **Low temperature** | 0.0-0.3 for factual tasks |
| **Structured output** | Constrain responses to valid formats |
| **Citation requirement** | "Cite which part of the context supports your answer" |
| **Validation layer** | Cross-reference the response with your data |

> You can never eliminate hallucinations completely, but you can reduce them significantly.

## Evaluating Prompt Quality

Create a **test suite**: inputs with expected outputs. Measure:

| Metric | Description |
|--------|-------------|
| **Correctness** | Does it produce the right answer? |
| **Format compliance** | Does it match the expected schema? |
| **Latency** | How long does it take? |
| **Token efficiency** | How many tokens does it use? |
| **Consistency** | Same input → same quality output? |

Version your prompts like code and track performance across versions.

## Quick Reference

| Concept | Description |
|---------|-------------|
| **Temperature** | 0 = deterministic, 1 = creative. Use 0-0.3 for factual, 0.7-1.0 for creative |
| **Token** | Basic unit of text processing — ~4 characters in English |
| **Grounding** | Connecting responses to factual, verifiable data sources |
| **Hallucination** | Model generates information that sounds correct but is fabricated |
| **Zero-shot** | Only instructions, no examples |
| **Few-shot** | Examples of desired behavior (usually 2-5) |
| **Prompt injection** | Malicious input that tricks the model into ignoring instructions |
| **Model fallback** | Use cheap model as default, escalate to expensive model when needed |
| **Semantic caching** | Cache by meaning, not exact query match — similar questions return cached responses |

---

[← Previous: OpenAI API](02-openai-api.md) | [Next: Semantic Kernel →](04-semantic-kernel.md) | [Back to index](README.md)
