# CLR e IL

## Common Language Runtime (CLR)

O **CLR (Common Language Runtime)** é um componente do .NET que gerencia a execução de aplicações. Ele é responsável por carregar e executar código escrito em várias linguagens .NET, incluindo C#, VB.NET, F# e outras.

## Como funciona

### 1. Compilação e Execução

- Quando escrevemos um programa em C#, ele é compilado em **Intermediate Language (IL)**, também chamado de **CIL (Common Intermediate Language)** ou **MSIL (Microsoft Intermediate Language)**. Esse código é independente de plataforma.
- O CLR usa um compilador **Just-In-Time (JIT)** para converter o código IL em código de máquina específico enquanto o programa roda.

```
Source Code → Language Compiler → MSIL Code + Metadata
                                        ↓
                               Just-In-Time Compiler → Native Code
```

### 2. Serviços fornecidos pelo CLR

- **Gerenciamento automático de memória** através do Garbage Collection, prevenindo memory leaks
- **Type safety** — garante que os tipos de dados são usados corretamente e com segurança
- **Verificação de segurança** — o CLR verifica o código IL buscando riscos de segurança antes de executá-lo

### 3. Integração entre linguagens (Cross-Language Integration)

O CLR permite que código de diferentes linguagens .NET (C#, VB.NET, F#) trabalhe junto de forma transparente através do **Common Type System (CTS)**.

## AOT (Ahead-of-Time Compilation)

A partir do .NET 7/8, existe a opção de **Native AOT**, que compila o código diretamente para código nativo no momento do build, eliminando a necessidade do JIT em runtime. Isso resulta em:

- Startup mais rápido
- Menor consumo de memória
- Porém com algumas limitações (reflexão limitada, por exemplo)

---

[← Anterior: Namespace](02-namespace.md) | [Voltar ao índice](README.md) | [Próximo: Tipos Numéricos →](04-tipos-numericos.md)
