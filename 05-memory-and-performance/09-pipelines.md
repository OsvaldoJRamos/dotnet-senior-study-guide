# System.IO.Pipelines

When you read bytes from a network or stream and try to parse messages out of them, you hit three problems immediately: data arrives in arbitrary chunks, individual messages can span multiple chunks, and a single chunk can contain partial data of the next message. `System.IO.Pipelines` is the abstraction Kestrel uses internally to solve those problems with zero-allocation parsing.

## The classic problem

A naive socket loop looks like this:

```csharp
byte[] buffer = new byte[1024];
int read = stream.Read(buffer, 0, 1024);
// You read 1024 bytes. But:
// - what if the message is 2000 bytes? You only have half.
// - what if the buffer contains 1.5 messages? Where does the next one start?
// - what if a multi-byte field straddles the buffer boundary?
```

You end up writing your own framing logic, allocation-heavy `byte[]` copies between reads, and probably reinventing buffering — badly.

## How Pipelines solves it

A pipeline separates the **producer** (whatever pulls bytes from the source) from the **consumer** (your parser). Both share an internal buffer the pipeline manages, with explicit semantics for "I consumed up to here" and "I examined up to here."

```csharp
using System.IO.Pipelines;

PipeReader reader = PipeReader.Create(stream);

while (true)
{
    ReadResult result = await reader.ReadAsync();
    ReadOnlySequence<byte> buffer = result.Buffer;

    while (TryParseMessage(ref buffer, out Message message))
    {
        Process(message);
    }

    // Tell the pipeline: I consumed up to buffer.Start (after parsed messages),
    // and I examined up to buffer.End (everything available).
    reader.AdvanceTo(buffer.Start, buffer.End);

    if (result.IsCompleted) break;
}

await reader.CompleteAsync();
```

The two-position `AdvanceTo(consumed, examined)` is the trick. It tells the pipeline:

- **consumed**: don't give me these bytes again — they're processed
- **examined**: I looked at everything up to here but didn't have enough to make progress; if you have more data, give me everything from `consumed` onward

If you only call `AdvanceTo(consumed)` (single-argument overload), the pipeline assumes consumed = examined, which can deadlock the parser if you didn't have enough data to commit.

## `PipeReader` API surface

The verified methods (per the official MS Learn `PipeReader` reference):

```csharp
abstract class PipeReader
{
    // Reading
    ValueTask<ReadResult> ReadAsync(CancellationToken cancellationToken = default);
    ValueTask<ReadResult> ReadAtLeastAsync(int minimumSize, CancellationToken cancellationToken = default);
    bool TryRead(out ReadResult result);

    // Two-position advance — the important one
    void AdvanceTo(SequencePosition consumed, SequencePosition examined);
    void AdvanceTo(SequencePosition consumed);

    // Lifecycle
    void Complete(Exception? exception = null);
    ValueTask CompleteAsync(Exception? exception = null);
    void CancelPendingRead();

    // Bridges
    Task CopyToAsync(Stream destination, CancellationToken cancellationToken = default);
    Task CopyToAsync(PipeWriter destination, CancellationToken cancellationToken = default);
    Stream AsStream(bool leaveOpen = false);

    // Factories
    static PipeReader Create(Stream stream, StreamPipeReaderOptions? options = null);
    static PipeReader Create(ReadOnlySequence<byte> sequence);
}
```

The `ReadResult` returned by `ReadAsync` has `Buffer` (a `ReadOnlySequence<byte>`), `IsCompleted` (producer finished), and `IsCanceled` (read was cancelled).

## `ReadOnlySequence<T>` — the buffer model

`ReadOnlySequence<T>` is what `ReadResult.Buffer` returns. It's a logical view over **multiple non-contiguous segments** of memory — because data from a socket arrives in chunks and forcing them to be contiguous would mean copying.

```csharp
ReadOnlySequence<byte> seq = result.Buffer;

if (seq.IsSingleSegment)
{
    // Fast path: everything in one Span
    ReadOnlySpan<byte> span = seq.FirstSpan;
    Process(span);
}
else
{
    // Slow path: walk segments
    foreach (ReadOnlyMemory<byte> segment in seq)
    {
        Process(segment.Span);
    }
}

// Slicing is cheap — just position math, no copy
ReadOnlySequence<byte> first100 = seq.Slice(0, 100);
SequencePosition mid = seq.GetPosition(50);
ReadOnlySequence<byte> after = seq.Slice(mid);
```

For parsing, `SequenceReader<T>` (a `ref struct`) gives you forward-only readers across the whole sequence:

```csharp
SequenceReader<byte> reader = new(buffer);

if (reader.TryReadBigEndian(out int length) &&
    reader.TryReadExact(length, out ReadOnlySequence<byte> payload))
{
    ProcessMessage(payload);
}
```

## `PipeWriter` — the producer side

```csharp
PipeWriter writer = PipeWriter.Create(stream);

// Ask for a writable span
Memory<byte> buffer = writer.GetMemory(sizeHint: 1024);
int written = WriteData(buffer.Span);
writer.Advance(written);     // commit the bytes you actually wrote
await writer.FlushAsync();    // backpressure point — waits if downstream is full
```

`GetMemory(N)` returns a buffer of **at least** N bytes (often more). You write into it, then call `Advance(M)` with the actual byte count.

## `IBufferWriter<T>` — the abstraction the JSON writer uses

`IBufferWriter<T>` is the interface `PipeWriter` implements. Many APIs accept it directly so you can write without intermediate buffers:

```csharp
public interface IBufferWriter<T>
{
    Memory<T> GetMemory(int sizeHint = 0);
    Span<T> GetSpan(int sizeHint = 0);
    void Advance(int count);
}
```

Implementations include:

- `PipeWriter` — write straight to the pipeline / network
- `ArrayBufferWriter<T>` — write to a growable array

`System.Text.Json`'s `Utf8JsonWriter` accepts an `IBufferWriter<byte>`. That means you can serialize JSON directly to the response pipeline with no intermediate `string` or `byte[]`:

```csharp
// In an ASP.NET Core endpoint
await using var writer = new Utf8JsonWriter(Response.BodyWriter);
JsonSerializer.Serialize(writer, dto);
await Response.BodyWriter.FlushAsync();
```

Zero string materialization, zero array copy, zero double-buffering.

## Where Pipelines fits

- **Custom TCP/UDP servers** — gRPC servers, RESP/Redis-compatible servers, MQTT brokers
- **Binary protocol parsers** — MessagePack, Protobuf, custom proprietary protocols
- **WebSocket frame handlers**
- **Anything where you need to parse a stream of bytes into typed messages with backpressure**

For HTTP, you don't need to learn this directly — Kestrel uses pipelines under the hood. But knowing how it works lets you read Kestrel source code, understand profiler stacks, and reason about HTTP/2 framing.

## What you don't get for free

- **Framing logic is still yours** — pipelines give you a buffer with good semantics; you write the parser that says "this is a complete message, here's where it ends"
- **Cancellation needs care** — `ReadAsync` accepts a `CancellationToken`, but you also need to wire `CompleteAsync` paths so the producer side knows the consumer is done
- **One reader per pipeline** — `PipeReader` is single-consumer; multiplex by separating pipelines, not by sharing one

> **Senior signal:** when an interviewer asks "how would you build a custom protocol server in .NET?" the textbook answer references `System.IO.Pipelines` + `ReadOnlySequence<byte>` + `SequenceReader<byte>` for parsing. Knowing this shows you've moved past `Stream.Read` patterns.

---

[← Previous: Diagnostics](08-diagnostics.md) | [Back to index](README.md) | [Next: Memory Model →](10-memory-model.md)
