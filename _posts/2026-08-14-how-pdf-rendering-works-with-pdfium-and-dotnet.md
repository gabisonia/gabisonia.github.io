---
layout: post
title: "How PDF Rendering Works With PDFium And .NET"
date: 2026-08-14 09:15:00 +0200
categories: dotnet pdf architecture
excerpt: "A practical look at PDF rendering, how PDFium turns a document page into pixels, and what PdfiumRaster adds for .NET applications."
---

In my previous post I wrote about [using named pipes in PdfiumRaster.Orchestrator](/dotnet/architecture/ipc/named-pipes-in-dotnet-a-practical-use-case/). The orchestrator starts worker processes and sends PDF rendering requests to them.

That explains how the processes communicate but skips one important question.

What is the worker actually doing with the PDF?

The short answer is that it uses [PdfiumRaster](https://github.com/gabisonia/PdfiumRaster) which uses PDFium to turn a PDF page into pixels. After that the pixels can be saved as PNG, JPEG, WebP or BMP.

This post is about that lower layer.

## The Use Case

Imagine an application where users upload PDF documents. The application needs to show a preview for every page.

The browser can display a PDF with its own viewer but that is not always enough. Maybe we need small thumbnails in a document list. Maybe another service needs a JPEG. Maybe an OCR pipeline needs a clean page image before it can recognize text.

The same need appears in many systems:

- create thumbnails for uploaded documents
- convert pages for OCR or computer vision
- show previews without a full PDF viewer
- generate images for document comparison
- export pages into another workflow

In all of these cases the input is a PDF and the useful output is a normal image.

It sounds like file conversion but there is quite a lot happening between those two files.

## A PDF Is Not An Image

A PDF page may look like an image on the screen but internally it is a document description.

PDF is a fixed-layout format. It can contain text with embedded fonts, vector paths, raster images, transparency, color spaces, annotations and many other objects. A page content stream contains drawing operations which describe how those objects should be painted.

This is closer to a small graphics program than a screenshot.

For example a page can roughly say:

```text
select this font
set the text size
move to this position
draw these characters
draw a line
place an image here
```

The real PDF syntax is more compact and complicated. Objects can refer to other objects. Streams can be compressed. Cross-reference data tells the reader where objects are stored. The format also changed over time so a parser has to understand several ways of representing the same kind of structure.

The [PDF Association description of the format](https://pdfa.org/about-us/the-portable-document-format/) explains the main purpose well: a PDF keeps the visual presentation independent from the software and operating system which created it.

That flexibility is good for documents. It also means that changing `.pdf` to `.png` is not possible. Something has to parse the document and execute the page drawing instructions.

## PDF Processing Is A Broad Term

People often say PDF processing when they mean very different jobs.

It can mean:

- reading text
- merging or splitting documents
- editing page objects
- filling forms
- adding signatures
- validating PDF/A
- rendering pages

PdfiumRaster is focused only on rendering. It can read basic page information and convert pages to images. It does not try to edit files or extract their text structure.

Keeping that boundary small was intentional. A rendering library already has enough responsibility around native handles, memory, pixel formats and image encoding.

## Rasterization

Rasterization is the step where a page description becomes a rectangular grid of pixels.

Text and vector paths do not have a fixed pixel size inside the PDF. They can be drawn at different resolutions. That is why the same PDF can look sharp both as a small preview and when printed.

Before rendering we have to choose an output size. This can be an explicit width and height or it can be calculated from DPI.

PDF page sizes are measured in points. There are 72 PDF points in one inch. The basic conversion is:

```text
pixels = points / 72 * DPI
```

An A4 page is roughly 595 by 842 points. At 96 DPI it becomes around 793 by 1123 pixels. At 300 DPI it becomes around 2479 by 3508 pixels.

This choice has a direct memory cost. A 2479 by 3508 BGRA bitmap needs about 35 MB before PNG or JPEG compression. That is only one page.

The source PDF can be 200 KB and still produce a large bitmap. Input file size does not tell us the output memory size.

This is one reason why DPI should be a real application decision. A web thumbnail normally does not need 300 DPI.

## Where PDFium Fits

[PDFium](https://pdfium.googlesource.com/pdfium/+/refs/heads/main/) is a native C++ PDF library from the Chromium project. It handles the difficult part: parsing the PDF and rendering its page content.

Its public API is a C interface. The normal rendering lifecycle looks roughly like this:

```text
initialize PDFium
    -> load document
        -> load page
            -> create bitmap
                -> render page into bitmap
            -> destroy bitmap
        -> close page
    -> close document
destroy PDFium
```

At the native level these operations use functions such as:

```c
FPDF_InitLibrary();
FPDF_LoadDocument(...);
FPDF_LoadPage(...);
FPDFBitmap_CreateEx(...);
FPDF_RenderPageBitmap(...);
FPDF_ClosePage(...);
FPDF_CloseDocument(...);
FPDF_DestroyLibrary();
```

The official [`fpdfview.h`](https://pdfium.googlesource.com/pdfium/+/refs/heads/main/public/fpdfview.h) is the main API for initializing the library, loading documents and rendering pages.

PDFium does not return a ready PNG file from `FPDF_RenderPageBitmap`. It paints into a bitmap buffer supplied by the caller. That buffer contains raw pixels.

This separation matters. PDF rendering and image encoding are different jobs.

## What PdfiumRaster Adds

Calling a native C API from .NET means more than adding a few P/Invoke declarations.

Every handle needs the correct lifetime. Managed memory passed to native code must stay in place. Stream callbacks must remain alive while PDFium can call them. Errors from native code need to become useful .NET exceptions.

PdfiumRaster owns this workflow and exposes a smaller .NET API for PDF-to-image conversion.

The simplest usage is one line:

```csharp
using PdfiumRaster;

PdfImageConverter.SavePng(
    "input.pdf",
    pageNumber: 1,
    "page-0001.png");
```

Behind that call the library:

1. Initializes PDFium.
2. Opens the PDF document.
3. Loads the requested page.
4. Calculates the output pixel dimensions.
5. Creates a bitmap buffer.
6. Asks PDFium to render into it.
7. Encodes the pixels as PNG.
8. Releases the page, document and native resources.

PDFium performs the PDF parsing and page rendering. PdfiumRaster manages the .NET-facing lifecycle and options. It writes BMP directly and uses SkiaSharp for PNG, JPEG and WebP encoding.

This is the main reason the library exists. The goal is not to hide what PDFium is doing. The goal is to make one focused workflow safe and practical from .NET.

## Controlling The Output

A preview and a printable image need different settings.

For a screen preview I can render at 96 DPI:

```csharp
var options = new PdfImageConversionOptions
{
    Format = PdfImageOutputFormat.Png,
    Render = PdfPageRenderOptions.ScreenPreview,
};

PdfImageConverter.SavePageNumber(
    "input.pdf",
    pageNumber: 1,
    "preview.png",
    options);
```

For another workflow I may choose an explicit width and keep the page aspect ratio:

```csharp
var options = new PdfImageConversionOptions
{
    Format = PdfImageOutputFormat.Jpeg,
    Encoding = new PdfImageEncodingOptions { Quality = 85 },
    Render = new PdfPageRenderOptions
    {
        Width = 1600,
        WithAspectRatio = true,
        Flags = PdfRenderFlags.Annot | PdfRenderFlags.LcdText,
    },
};
```

Rendering flags are also important. Annotations are separate PDF objects so they need to be included when the expected image should look like a normal viewer. Anti-aliasing affects how smooth text and paths look. The background matters because PDF pages can contain transparency.

These are not only image encoder settings. They change how PDFium paints the page.

## Input Is Not Always A File Path

In a web application the PDF may come from object storage, a database or an HTTP upload. PdfiumRaster accepts paths, byte arrays, Base64 strings and streams.

The choice affects memory.

A file path is the simplest option for a large document. A seekable stream also works well because PDFium can request blocks from different positions. PdfiumRaster provides the native random-access callback without copying the complete stream into another byte array.

A non-seekable stream cannot move backwards. PDFium needs random access so the library has to buffer that input first. A `byte[]` and a Base64 string also keep the complete PDF in managed memory.

The practical rule is simple: use a path or seekable stream for large documents when possible.

## Reusing An Open Document

Opening and parsing the same PDF for every page adds unnecessary work.

For a multi-page export `PdfRenderSession` keeps the document open. It can also reuse its current page and render buffer.

```csharp
using var session = PdfRenderSession.Open("input.pdf");

for (var pageIndex = 0; pageIndex < session.PageCount; pageIndex++)
{
    session.SavePage(
        pageIndex,
        $"images/page-{pageIndex + 1:D4}.png",
        new PdfImageConversionOptions
        {
            Format = PdfImageOutputFormat.Png,
            Render = PdfPageRenderOptions.ScreenPreview,
        });
}
```

This is better than opening the same file for every page. It also makes resource ownership clear. The session owns the native document until it is disposed.

## The Threading Limitation

PDFium has one important rule for this story. Its public API is not thread-safe. The official header says calls must happen on one thread or the embedder must ensure only one PDFium call happens at a time.

PdfiumRaster uses a process-wide lock around native calls. This keeps document and rendering operations safe. Image encoding can still overlap but two PDFium renders do not run at the same time inside one process.

This is where the previous named-pipes post connects.

When I needed real parallel rendering I could not solve it by starting more tasks. Every task still reached the same process-wide PDFium lock. The concurrency boundary had to become the operating-system process.

Each PdfiumRaster.Orchestrator worker has its own PDFium runtime. Named pipes carry requests to those workers. The process also gives crash isolation and allows a hard timeout to terminate stuck native work.

The complete picture looks like this:

```text
application
    -> PdfiumRaster.Orchestrator
        -> named pipe
            -> worker process
                -> PdfiumRaster
                    -> PDFium
                        -> BGRA pixels
                            -> PNG, JPEG, WebP or BMP
```

PdfiumRaster solves the rendering workflow inside one process. PdfiumRaster.Orchestrator solves scheduling and isolation across several processes.

They are separate libraries because they solve separate problems. The second one only makes sense after understanding the first one.
