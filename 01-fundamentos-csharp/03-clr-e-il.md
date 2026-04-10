# CLR and IL

## Common Language Runtime (CLR)

The **CLR (Common Language Runtime)** is a .NET component that manages application execution. It is responsible for loading and executing code written in various .NET languages, including C#, VB.NET, F#, and others.

## How it works

### 1. Compilation and Execution

- When we write a program in C#, it is compiled into **Intermediate Language (IL)**, also called **CIL (Common Intermediate Language)** or **MSIL (Microsoft Intermediate Language)**. This code is platform-independent.
- The CLR uses a **Just-In-Time (JIT)** compiler to convert IL code into platform-specific machine code while the program runs.

```
Source Code → Language Compiler → MSIL Code + Metadata
                                        ↓
                               Just-In-Time Compiler → Native Code
```

### 2. Services provided by the CLR

- **Automatic memory management** through Garbage Collection, preventing memory leaks
- **Type safety** — ensures that data types are used correctly and safely
- **Security verification** — the CLR verifies IL code for security risks before executing it

### 3. Cross-Language Integration

The CLR allows code from different .NET languages (C#, VB.NET, F#) to work together seamlessly through the **Common Type System (CTS)**.

## AOT (Ahead-of-Time Compilation)

Starting with .NET 7/8, there is the **Native AOT** option, which compiles code directly to native code at build time, eliminating the need for JIT at runtime. This results in:

- Faster startup
- Lower memory consumption
- However, with some limitations (limited reflection, for example)

---

[← Previous: Namespace](02-namespace.md) | [Back to index](README.md) | [Next: Numeric Types →](04-tipos-numericos.md)
