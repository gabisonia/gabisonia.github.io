---
layout: post
title: "Named Pipes in .NET: A Practical Use Case"
date: 2026-08-14 09:00:00 +0200
categories: dotnet architecture ipc
excerpt: "How named pipes became the communication layer between isolated PDF rendering workers in PdfiumRaster.Orchestrator."
---

I recently built [PdfiumRaster.Orchestrator](https://github.com/gabisonia/PdfiumRaster.Orchestrator). It runs PDF rendering in separate worker processes and communicates with them through named pipes.

Named pipes were not the starting point. The starting point was a limitation in PDFium.

PDFium has process-global state and its public API is not thread-safe. [PdfiumRaster](https://github.com/gabisonia/PdfiumRaster) protects native calls with a process-wide lock. This makes it safe but also means that adding more tasks inside the same process does not give true parallel rendering.

If two tasks call PDFium at the same time one of them still waits for the lock.

The way around this was to use multiple processes. Each process gets its own PDFium runtime and its own native lock. Once I had worker processes I needed a simple way for the main application and workers to talk.

This is where named pipes became useful.

## What Is A Named Pipe?

A named pipe is an operating-system mechanism for communication between processes.

From .NET it looks like a stream. One side writes bytes and the other side reads them. Unlike an anonymous pipe it has a name which allows another process to connect to it.

.NET provides two main types:

- `NamedPipeServerStream` creates and owns the pipe
- `NamedPipeClientStream` connects to an existing pipe

The words server and client only describe who creates the pipe and who connects. They do not mean there is a web server or a remote network call.

In this project everything stays on the same machine.

```text
application
    |
    |-- named pipe <--> worker 1 <--> PDFium
    |-- named pipe <--> worker 2 <--> PDFium
    `-- named pipe <--> worker 3 <--> PDFium
```

Each worker has one pipe and handles one request at a time. Different workers can render at the same time because they are separate processes.

## Why Not HTTP?

HTTP would work but it would add things I did not need.

There would be a port to manage. I would need startup and readiness logic around that port. The worker would start looking like a service even though it lives and dies together with the application.

Named pipes fit the problem better:

- communication is local
- both directions use one connection
- reads and writes can be asynchronous
- no port or network endpoint is needed
- the connection lifetime can follow the worker lifetime

This does not mean named pipes are always faster or always better. If workers need to run on other machines then a network protocol is the correct direction. In my case remote communication was not a requirement.

## The Application Is The Pipe Server

This part can feel backwards at first.

The main library is the named-pipe server. It creates the pipe before starting a worker. The worker process is the pipe client and connects back to it.

The server side is created like this in a simplified form:

```csharp
var pipe = new NamedPipeServerStream(
    pipeName,
    PipeDirection.InOut,
    maxNumberOfServerInstances: 1,
    PipeTransmissionMode.Byte,
    PipeOptions.Asynchronous | PipeOptions.CurrentUserOnly);

StartWorkerProcess(pipeName);
await pipe.WaitForConnectionAsync(cancellationToken);
```

The worker connects to the local machine:

```csharp
await using var pipe = new NamedPipeClientStream(
    ".",
    pipeName,
    PipeDirection.InOut,
    PipeOptions.Asynchronous | PipeOptions.CurrentUserOnly);

await pipe.ConnectAsync(cancellationToken);
```

Creating the server first is important. If the process starts first it can try to connect before the pipe exists. That creates a startup race which is annoying to reproduce and even more annoying to debug.

Every worker gets a unique pipe name. The pipe is persistent for that worker. I do not create a new connection for every rendered page.

## A Pipe Is Only A Stream Of Bytes

After the connection works another problem appears. A pipe does not know what a request or response is. It only carries bytes.

If I write two messages there is no guarantee that the other side will receive them in two matching reads. One read can return part of a message. It can also return bytes that belong to more than one write.

The application protocol needs to define its own message boundaries.

PdfiumRaster.Orchestrator uses a small frame format:

```text
4-byte frame length | 1-byte message type | payload
```

The message type tells the receiver what follows. Examples are `Request`, `InputChunk`, `BitmapHeader`, `OutputChunk`, `Complete` and `Error`.

Writing the header is straightforward:

```csharp
BinaryPrimitives.WriteInt32LittleEndian(
    header.AsSpan(0, sizeof(int)),
    payload.Length + 1);

header[sizeof(int)] = (byte)message;

await pipe.WriteAsync(header, cancellationToken);
await pipe.WriteAsync(payload, cancellationToken);
```

Reading needs more care. A single `ReadAsync` is not enough. The code must continue until the complete header or payload is received.

```csharp
private static async Task ReadExactlyAsync(
    Stream stream,
    byte[] buffer,
    CancellationToken cancellationToken)
{
    var offset = 0;

    while (offset < buffer.Length)
    {
        var read = await stream.ReadAsync(
            buffer.AsMemory(offset),
            cancellationToken);

        if (read == 0)
        {
            throw new EndOfStreamException();
        }

        offset += read;
    }
}
```

This is one of the main practical lessons. Named pipes make the transport simple but they do not remove protocol design.

Lengths must be validated before allocating memory. Message order must be checked. An unexpected end of stream must be treated as a broken worker and not as a valid empty response.

## Handshake Before Work

Starting a process and seeing a pipe connection is not enough to trust that everything is ready.

The orchestrator generates a random token for every worker. It passes the pipe name as a process argument and the token through an environment variable. After connecting the worker sends a `Hello` message with the protocol version and token.

The application checks both values and replies with `Ready`.

```text
orchestrator                 worker
     |                          |
     |     start process        |
     |------------------------->|
     |                          |
     |        Hello             |
     |<-------------------------|
     |        Ready             |
     |------------------------->|
     |                          |
     |        Request           |
     |------------------------->|
```

The protocol version prevents an incompatible worker and library from silently talking to each other. The token makes sure the connected process is the child that was just started.

`CurrentUserOnly` also restricts the pipe to the current operating-system user. These checks are useful but the worker is still not a security sandbox. It runs with the same identity and file access as the application.

## Moving A PDF Through The Pipe

PDF input can arrive as a path or as content.

For a path input only request metadata and the full path cross the pipe. The worker opens the file directly. This is the best option for large PDFs because there is no reason to copy the complete document through the pipe.

For a `byte[]` or `Stream` input the content has to cross the process boundary. The orchestrator sends it in 64 KiB `InputChunk` frames and finishes with `InputEnd`.

The worker writes those chunks to a private temporary file before opening it with PDFium. PDF rendering needs random access so keeping the input as a forward-only pipe stream would not be enough.

Results follow the same idea. Bitmap pixels and encoded stream output travel in chunks. If the caller asks to save an image to a path then the worker writes it directly and only the result status comes back.

This is an important design choice with IPC. Do not move large data only because you can. Sometimes passing a path is enough.

## Hard Timeouts And Crashes

The process boundary solved more than parallelism.

Native rendering cannot always be stopped safely with a cancellation token. If a worker hangs then the orchestrator can kill that process. The active request fails with a timeout and a replacement worker is started. Other workers continue with their own requests.

The same rule applies when a worker crashes or sends invalid protocol data. The pipe closes and only that worker slot is replaced.

This also explains why the pipe belongs to one worker. Its lifetime is the same as the process it represents.

There is one less obvious case. If a streaming response is stopped in the middle then unread frames can remain in the pipe. Reusing that connection for a new request could put the protocol out of alignment. The safe option is to replace the worker and start with a clean pipe.

Failed requests are not retried automatically. Rendering can be expensive and some requests write files. A hidden retry could repeat work or duplicate side effects.

## Tradeoffs I Would Keep In Mind

Named pipes worked well here but they are not free.

Every worker has its own process and native runtime so it uses more memory. Data still needs serialization. Large in-memory inputs and outputs have transfer cost. The protocol also becomes code that needs tests for wrong lengths, invalid messages, startup failures and disconnected workers.

I would consider named pipes when:

- two processes run on the same machine
- a child process needs to follow the application lifetime
- low-level stream communication is enough
- process isolation matters
- opening a network service would add no value

I would not use them for communication across machines or as a security boundary.

For PdfiumRaster.Orchestrator the choice came from the problem. Multiple processes gave PDFium real parallelism and crash isolation. Named pipes gave those processes a small local communication channel.

That is usually how an infrastructure choice should happen. Start with the limitation and then choose the smallest thing that solves it.
