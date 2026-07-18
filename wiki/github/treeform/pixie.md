# treeform/pixie

> A full-featured 2D graphics library written entirely in Nim — Cairo/Skia
> territory with CPU rasterization, codecs, and text layout in one package.

[GitHub repo](https://github.com/treeform/pixie) ·
[API reference](https://treeform.github.io/pixie) ·
[License: MIT](https://github.com/treeform/pixie/blob/master/LICENSE)

## Overview

Pixie is a pure-Nim 2D graphics library covering CPU rasterization, image
codecs, vector paths, paints, masking, blend modes, and text typesetting[^1].
It is a Nim-native answer to Cairo and Skia: no C bindings, no FreeType, no
HarfBuzz — the whole stack, including the TTF/OTF font parser and JPEG
decoder, is implemented in Nim by treeform (Andre von Houck) and guzba (Ryan
Oldenburg), who co-author most of its dependency ecosystem.

At 806 stars it is one of the most visible pure-Nim libraries — large for
the Nim ecosystem, small in absolute terms. The tradeoff to understand up
front: you get a self-contained, easy-to-cross-compile graphics stack with
zero system dependencies, but the implementation surface (fonts, codecs,
SVG) is a from-scratch subset of what battle-tested C/C++ libraries cover.
Development slowed to a trickle from 2023 through mid-2025, then resumed:
5.1.0 (2025-09), 6.0.0 (2026-04), 6.1.0 (2026-05), with pushes as recent as
July 2026[^2] — active again, but on a two-person bus factor.

## Getting Started

```bash
nimble install pixie
```

```nim
import pixie

let image = newImage(200, 200)
image.fill(rgba(255, 255, 255, 255))

let ctx = newContext(image)
ctx.fillStyle = rgba(255, 0, 0, 255)
ctx.fillRoundedRect(rect(vec2(50, 50), vec2(100, 100)), 25.0)

image.writeFile("rounded_rect.png")
```

The `Context` type deliberately mirrors the HTML Canvas 2D API, so browser
canvas knowledge transfers directly. The lower-level `Image`/`Path`/`Paint`
APIs accept raw SVG path strings (the full `M`/`L`/`C`/`A`/`z` set)[^1].

## Architecture / How It Works

Pixie is a **CPU rasterizer**: everything renders into an `Image` (a plain
RGBA pixel buffer) on the CPU, with anti-aliased coverage computed per
scanline and both even-odd and non-zero winding rules supported. There is no
GPU path inside pixie itself; the sanctioned realtime route is **Boxy**, a
companion library that uploads pixie-rendered layers to the GPU and
composites them at interactive framerates[^3].

The stack decomposes into treeform/guzba micro-libraries:

- **vmath** — vectors and matrices (`vec2`, `translate`, `scale`).
- **chroma** — color types and parsing (`parseHtmlColor`).
- **zippy** — deflate/zlib in Nim, used by the PNG codec.
- **bumpy** — 2D geometry/collision primitives.
- **nimsimd** — SIMD intrinsics; hot loops (blends, blurs, fills) have
  SSE2/AVX and NEON variants selected at compile time[^4].

Codecs are asymmetric by design: PNG, BMP, QOI, and PPM read and write;
JPEG, GIF, and SVG are read-only[^1]. SVG support is an importer for a
useful subset (paths, shapes, transforms, fills/strokes), not a conformant
renderer — icons work, complex documents (filters, CSS) may not.

Text is pixie's most ambitious subsystem: it parses TTF/OTF/SVG fonts and
does its own typesetting, including multi-font rich-text spans, alignment,
and wrapping. Blending and masking follow the Photoshop/Figma model — 16
blend modes (Multiply, Color Dodge, Hue, Luminosity, ...) plus mask modes
(Subtract, Intersect, Exclude), which makes pixie unusually well suited to
rendering design-tool documents on a server.

## Production Notes

- **Compile with `-d:release`.** Nim debug builds skip optimization and add
  runtime checks; rasterization can be an order of magnitude slower. This is
  the number-one "pixie is slow" false alarm.
- **CPU-only means CPU-bound.** Large canvases with heavy blurs/shadows cost
  real milliseconds. For animation or games, render layers with pixie and
  composite with Boxy on the GPU rather than re-rasterizing per frame[^3].
- **Text shaping is basic.** No HarfBuzz underneath: Latin-script
  typesetting is good, but complex-script shaping (Arabic joining, Indic
  reordering) and full bidi are not what this engine is built for. Validate
  with your target languages before committing.
- **No JPEG encode.** Pipelines that must emit JPEG need a separate encoder;
  pixie only decodes it.
- **Version churn.** Majors have reworked the paint/mask APIs; 4.x-era
  tutorial code frequently fails on 5.x/6.x. Pin the version in your
  `.nimble` file and read release notes when bumping majors[^2].
- **Maintenance cadence.** The 2023–2025 gap (only 5.0.7 in two years) is
  worth knowing: alive in 2026, but a two-person side project, not a
  foundation-backed engine. Budget for reading the (readable) source.

## When to Use / When Not

**Use when:**
- You write Nim and need server-side or batch image generation (charts,
  avatars, OG images, thumbnails) with zero system dependencies.
- You render design-tool-style documents: the Figma-like blend/mask model
  and rich-text spans map directly.
- You want one static binary that parses fonts, rasterizes vectors, and
  encodes PNG — cross-compilation without a C dependency chain.
- You are building a Nim GUI/game and pair it with Boxy for GPU compositing.

**Avoid when:**
- You need conformant SVG rendering or full CSS-level text layout — use a
  browser engine or resvg-class renderer instead.
- Your text is multilingual with complex scripts; shaping is the bottleneck.
- You need GPU-accelerated vector graphics in one library (see vello, skia).
- You are not in Nim and not using the Python bindings: the benefit largely
  evaporates through FFI.

## Alternatives

- google/skia — the industrial option (GPU backends, full text shaping,
  browser-grade codecs); use when coverage beats build simplicity.
- RazrFalcon/tiny-skia — closest analog in Rust: a CPU-only Skia subset;
  use it in Rust projects with the same "small, no GPU" philosophy.
- linebender/vello — use when you need GPU compute-shader vector rendering
  rather than CPU rasterization.
- python-pillow/Pillow — use for Python-native image manipulation when you
  don't need pixie's vector/typesetting features.
- fogleman/gg — the Go equivalent for simple canvas-style 2D drawing.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.0.x | 2020-11 | Repo created; rapid early iteration[^2]. |
| 1.0.0 | 2021-02 | First stable release. |
| 1.2.0 | 2021-05 | Text support added. |
| 3.0.0 | 2021-10 | Python bindings, many fixes[^5]. |
| 4.0.0 | 2022-02 | API rework era; monthly 4.x releases through mid-2022. |
| 5.0.0 | 2022-08 | Major performance improvements[^4]. |
| 5.0.7 | 2024-04 | Lone release in the 2023–2025 slow period. |
| 5.1.0 | 2025-09 | Maintenance resumes. |
| 6.0.0 | 2026-04 | Breaking major release. |
| 6.1.0 | 2026-05 | Current release line[^2]. |

## References

[^1]: Pixie README — feature matrix, codecs, blend/mask modes, SVG path
    support. https://github.com/treeform/pixie
[^2]: GitHub releases, treeform/pixie.
    https://github.com/treeform/pixie/releases
[^3]: Boxy — GPU compositing companion to pixie.
    https://github.com/treeform/boxy
[^4]: "Pixie 5.0 performance improvements" (guzba/treeform).
    https://www.youtube.com/watch?v=Did21OYIrGI
[^5]: Release 3.0.0 — "Python Bindings and Many Fixes".
    https://github.com/treeform/pixie/releases/tag/3.0.0

## Tags

nim, 2d-graphics, rasterization, vector-graphics, image-processing,
text-rendering, svg, png, cpu-rendering, simd, graphics-library
