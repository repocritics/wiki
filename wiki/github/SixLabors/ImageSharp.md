# SixLabors/ImageSharp

> A fully managed, cross-platform 2D image processing library for .NET — and the de facto replacement for the Windows-only System.Drawing.

[GitHub repo](https://github.com/SixLabors/ImageSharp) ·
[Official website](https://sixlabors.com/products/imagesharp/) ·
[License: Six Labors Split License 1.0](https://github.com/SixLabors/ImageSharp/blob/main/LICENSE) (source-available, not OSI-approved; GitHub reports `NOASSERTION`)

## Overview

ImageSharp is a pure-C#, dependency-free image processing library for .NET, first opened in 2016 and shipped stable as 1.0 in early 2020 after a long beta[^1]. It decodes, edits, and encodes the common raster formats — JPEG, PNG, GIF, BMP, TIFF, WebP, plus PBM/TGA/QOI — with no native libraries, no P/Invoke, and no GDI+. That "fully managed" design is the whole point: the same code runs identically on Windows, Linux, macOS, containers, and serverless, which is exactly where the old `System.Drawing` stack fell apart.

Its adoption curve is tied to a Microsoft decision. In .NET 6 (2021), `System.Drawing.Common` was made Windows-only and throws on Linux/macOS by default[^2]. That broke a large amount of server-side image code overnight and pushed the ecosystem toward two managed-friendly options: ImageSharp and SkiaSharp. ImageSharp won the "idiomatic .NET, no native binary" niche.

The defining tension is not technical but legal. Through v1 and v2 ImageSharp was Apache-2.0. Starting with **v3.0 (2023) the project moved to the Six Labors Split License**: free for individuals, OSS projects, and small companies, but requiring a paid commercial license above a revenue threshold[^3]. GitHub cannot classify it, so the repo shows `NOASSERTION`. This means "is ImageSharp free for us?" is a question your finance/legal team answers, not your engineers — and it is the single most important thing to understand before adopting it.

## Getting Started

```bash
dotnet add package SixLabors.ImageSharp
```

```csharp
using SixLabors.ImageSharp;
using SixLabors.ImageSharp.Processing;

// Load is format-agnostic; the decoder is picked from the file's magic bytes.
using Image image = Image.Load("input.jpg");

image.Mutate(ctx => ctx
    .Resize(image.Width / 2, image.Height / 2)
    .Grayscale());

image.Save("output.png"); // encoder inferred from the .png extension
```

For drawing shapes, paths, and text you additionally need the companion package `SixLabors.ImageSharp.Drawing` (plus `SixLabors.Fonts`); that functionality is deliberately not in the core package.

## Architecture / How It Works

The central type is `Image<TPixel>`, generic over a pixel format (`Rgba32`, `Rgb24`, `L8`, etc.), with a non-generic `Image` façade for format-agnostic work. Pixels are stored in pooled, contiguous memory managed by a configurable `MemoryAllocator`, so a decode does not translate to one naive `new byte[]` — buffers are rented and returned to reduce GC pressure on high-throughput services.

Processing is a pipeline. `image.Mutate(ctx => ...)` applies operations in place; `image.Clone(ctx => ...)` produces a modified copy. Each operation (`Resize`, `Crop`, `Gaussian­Blur`, `Grayscale`, color-matrix filters, quantization) is an `IImageProcessor` implementing a row-parallel `Apply`. This is where ImageSharp spends its performance budget: heavy use of `System.Numerics` vectors and hardware intrinsics (SSE/AVX/NEON) to close the gap with native libraries that a managed implementation would otherwise lose.

Codecs are pluggable via `Configuration`. Each format ships a decoder/encoder pair registered in `Configuration.Default`; you can add, remove, or replace them, and pass per-call `DecoderOptions` (target size, max frame count, segment limits) that also serve as the front line against decompression-bomb inputs.

The project is deliberately split into separate packages rather than one monolith:

- **SixLabors.ImageSharp** — decode/process/encode core.
- **SixLabors.ImageSharp.Drawing** — vector drawing, shapes, text rasterization (separate release cadence, separately versioned).
- **SixLabors.Fonts** — font loading and text layout, used by Drawing.
- **SixLabors.ImageSharp.Web** — ASP.NET Core middleware for on-the-fly resize/format via URL, with caching providers.

This modularity keeps the core small but means "ImageSharp can draw text on an image" requires assembling two or three packages whose versions must stay compatible.

## Production Notes

**The license is the first production decision, not the last.** v3+ under the Six Labors Split License requires a commercial license above the revenue threshold[^3]. Teams that cannot or will not license commonly pin to the **2.x line, which remains Apache-2.0** — but 2.x no longer receives features and its security-fix horizon is finite. Treat "which major version am I allowed to run" as a compliance item tracked in your SBOM.

**Memory, not CPU, is usually the ceiling.** A decoded image is width × height × bytes-per-pixel held in memory; a 8000×6000 RGBA image is ~192 MB before any processing copy. Under concurrency this dominates. Tune `Configuration.Default.MemoryAllocator`, cap concurrent decodes, and call `MemoryAllocator.Default.ReleaseRetainedResources()` if pooled buffers accumulate. Set `DecoderOptions` limits on any endpoint that accepts user-uploaded images — otherwise a small malicious file can request a huge allocation.

**`Image` instances are not thread-safe.** Do not share one across threads; the pipeline parallelizes internally across rows, so per-image concurrency is already handled for you. One image per request/task.

**Performance is competitive but rarely the fastest.** For raw bulk decode/encode throughput, native-backed SkiaSharp and Magick.NET frequently beat ImageSharp; ImageSharp wins on deployment simplicity (no native binary to ship per RID), determinism across platforms, and pure-managed sandboxing. Benchmark your own hot path — resize-heavy thumbnailing and codec-bound batch jobs can land very differently.

**Upgrade friction is real across majors.** 1→2 and especially 2→3 changed public API surface (decoder/encoder registration, options types) and, for 3, the license. The core and the `.Drawing` companion version independently, so a core bump can leave you hunting a matching Drawing release. Read the release notes before bumping a major.

## When to Use / When Not

**Use when:**
- You need cross-platform server-side imaging in .NET without shipping native binaries (containers, serverless, Linux hosts).
- You're replacing `System.Drawing.Common` after its .NET 6 Windows-only change[^2].
- You want idiomatic, allocation-aware C# with a fluent processing pipeline.
- You can either stay within the free tier of the Six Labors Split License or budget for a commercial license.

**Avoid when:**
- Your organization is above the revenue threshold and won't pay for a license, and you need current features (pin to Apache-2.0 v2, or use an alternative).
- You need the widest possible format coverage or exotic codecs — Magick.NET (ImageMagick) covers far more.
- You need maximum raw decode/encode throughput and can accept native dependencies — SkiaSharp or libvips are often faster.
- You need heavy computer-vision operations rather than 2D graphics — that's OpenCV territory.

## Alternatives

- mono/SkiaSharp — Google Skia bindings; native, fast, MIT-licensed; use when you want speed and can ship per-platform native assets.
- dlemstra/Magick.NET — ImageMagick bindings; the broadest format and operation coverage; use when you need formats ImageSharp lacks.
- kleisauke/net-vips — libvips bindings; low-memory streaming pipeline; use for very large images or high-volume thumbnailing.
- dotnet/runtime (System.Drawing.Common) — Microsoft's GDI+ wrapper; Windows-only since .NET 6; use only for Windows-only legacy code.
- Emgu.CV / shimat/opencvsharp — OpenCV bindings; use when the task is computer vision, not 2D image editing.

## History

| Version | Date | Notes |
|---------|------|-------|
| beta | 2016-2019 | Long public beta; API churned heavily before stabilizing[^1]. |
| 1.0.0 | 2020-01 | First stable release; Apache-2.0 licensed[^1]. |
| 2.0.0 | 2022-03 | Performance and API rework; still Apache-2.0. |
| 3.0.0 | 2023 | Moved to the Six Labors Split License; dropped older target frameworks, targets modern .NET (6/8)[^3]. |

## References

[^1]: SixLabors/ImageSharp releases and repository history — created 2016-10-28, first stable 1.0 in 2020. https://github.com/SixLabors/ImageSharp/releases
[^2]: .NET docs, "System.Drawing.Common only supported on Windows" breaking change (.NET 6). https://learn.microsoft.com/en-us/dotnet/core/compatibility/core-libraries/6.0/system-drawing-common-windows-only
[^3]: Six Labors Split License, Version 1.0 — free for individuals, OSS, and small companies; commercial license required above the revenue threshold. https://github.com/SixLabors/ImageSharp/blob/main/LICENSE

## Tags

csharp, dotnet, image-processing, graphics, jpeg, png, webp, cross-platform, source-available, managed, imaging
