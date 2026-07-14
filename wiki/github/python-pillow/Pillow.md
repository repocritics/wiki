# python-pillow/Pillow

> The maintained fork of the Python Imaging Library — the default way Python reads, writes, and manipulates raster images.

[GitHub repo](https://github.com/python-pillow/Pillow) ·
[Official website](https://python-pillow.github.io) ·
[License: HPND](https://github.com/python-pillow/Pillow/blob/main/LICENSE)

## Overview

Pillow is a fork of PIL, the Python Imaging Library written by Fredrik Lundh and contributors. PIL stalled after its last release (1.1.7) in 2009; Jeffrey "Alex" Clark started the Pillow fork in 2010 to keep it installable on modern Python and to fix the packaging that had made PIL notoriously hard to build[^1]. Pillow long ago superseded the original — it is the `import PIL` that virtually every Python image workflow depends on, directly or transitively, and one of the most-downloaded packages on PyPI.

The library's job is deliberately narrow: decode and encode a very wide set of file formats, hold pixels in a compact in-memory representation, and offer a pragmatic set of per-pixel and geometric operations (resize, crop, filter, draw, composite). It is not a computer-vision library and not a numerical array framework — it is the I/O and light-manipulation layer that sits underneath those. That narrow scope is the source of both its ubiquity and its recurring security exposure: most of the real work happens in C, calling into third-party codec libraries whose bugs become Pillow's CVEs.

The defining tension is age. Pillow carries an API and internal model designed in the early 1990s (immutable-ish `Image` objects, string "modes" like `"RGB"` and `"L"`, palette images, lazy decoding). That model is stable and well understood, but it predates NumPy, predates Unicode-by-default, and predates any expectation of thread-safety or memory-safety in image decoders.

## Getting Started

```bash
pip install Pillow    # binary wheels bundle libjpeg/zlib/libtiff/etc.
```

```python
from PIL import Image, ImageOps, ImageFilter

im = Image.open("input.jpg")          # lazy — header only, not decoded yet
im = ImageOps.exif_transpose(im)      # apply EXIF orientation (NOT automatic)

im = im.convert("RGB")                # normalize mode
thumb = im.copy()
thumb.thumbnail((512, 512))           # in-place, preserves aspect ratio
thumb = thumb.filter(ImageFilter.SHARPEN)

thumb.save("output.webp", quality=82) # format inferred from extension
```

## Architecture / How It Works

Pillow is a thin Python layer over a C core (`_imaging`, compiled from `src/libImaging`). The Python `Image.Image` object is a handle; the actual pixel buffer lives in C and is only materialized when needed.

- **Lazy loading.** `Image.open()` reads just enough to identify the format and populate `size`/`mode`. Decoding is deferred until `.load()`, a pixel access, or an operation forces it. This makes `open()` cheap but means an exception can surface far from the `open()` call.
- **Plugins.** Each format is a plugin module (`JpegImagePlugin`, `PngImagePlugin`, `TiffImagePlugin`, …) registered at import time. `Image.open` sniffs magic bytes and dispatches. The actual encode/decode is done by bundled or system C libraries: libjpeg-turbo, zlib, libtiff, libwebp, OpenJPEG, libfreetype (text), and LittleCMS (color management).
- **Modes.** Pixel layout is a string mode: `"1"` (bilevel), `"L"` (8-bit gray), `"P"` (palette), `"RGB"`, `"RGBA"`, `"CMYK"`, `"YCbCr"`, `"I"`/`"I;16"` (integer), `"F"` (float), and others. Many operations are only defined for a subset of modes, so `convert()` is a frequent and easy-to-forget prerequisite.
- **NumPy interface.** `numpy.asarray(im)` and `Image.fromarray(arr)` bridge to the array world via the buffer protocol. This is the seam where Pillow meets OpenCV, scikit-image, and PyTorch/TensorFlow data loaders.

Because the heavy lifting is in C, some operations release the GIL, but the `Image` object itself is not designed for concurrent mutation. Treat an `Image` as owned by one thread at a time.

## Production Notes

- **`Image.ANTIALIAS` is gone.** Removed in Pillow 10 (2023) after long deprecation. Use `Image.Resampling.LANCZOS`. Similarly `im.textsize()` was removed in favor of `ImageDraw.textbbox`/`textlength`. Code and tutorials from before 2023 break on modern Pillow — one of the most common upgrade failures[^2].
- **EXIF orientation is not applied on load.** Phone photos will appear rotated unless you call `ImageOps.exif_transpose()`. This surprises nearly everyone once.
- **Decompression-bomb protection.** Pillow raises `DecompressionBombWarning` past ~89 million pixels and errors past twice that, guarding against malicious files that decode to enormous buffers. Legitimate large images need `Image.MAX_IMAGE_PIXELS` raised or set to `None`[^3].
- **Security surface.** A large share of Pillow's historical CVEs are memory-safety bugs in the underlying C decoders (libtiff especially) triggered by crafted input. If you decode untrusted images, pin a current Pillow, keep the bundled libs fresh (use the wheels, not an old distro libjpeg), and consider sandboxing. OSS-Fuzz runs against Pillow continuously[^4].
- **Performance.** Pillow's resize/filter code is competent but not vectorized for the newest SIMD. `uploadcare/pillow-simd` is a drop-in fork with SSE4/AVX2 kernels that is several times faster on resize-heavy pipelines — but it shadows the `PIL` namespace and lags upstream releases, so it is a deliberate operational choice, not a free win. For very large images or low-memory throughput, libvips (pyvips) streams tiles instead of loading whole rasters and outperforms Pillow substantially.
- **Palette (`"P"`) mode footguns.** Operations silently assume RGB; convert to `"RGB"`/`"RGBA"` before drawing, compositing, or resizing unless you specifically want palette behavior.
- **Multi-frame formats** (GIF, multipage TIFF, APNG) require `seek()`/`tell()` or `ImageSequence.Iterator`; a naive `open().save()` keeps only the first frame.

## When to Use / When Not

**Use when:**
- You need to read/write image files across many formats and do light manipulation (resize, crop, thumbnail, watermark, format conversion).
- You want the ecosystem-default that every other library already interoperates with.
- You need simple text/shape drawing (`ImageDraw`, `ImageFont`) without pulling in a rendering stack.

**Avoid (or supplement) when:**
- You need computer vision — feature detection, contours, transforms, video — reach for OpenCV.
- You process very large images or need maximum resize throughput — libvips/pyvips or pillow-simd.
- You do scientific/array-based image analysis — scikit-image on NumPy arrays is the better model.
- You are decoding untrusted input at scale and cannot patch promptly — the C decoder surface is a real risk.

## Alternatives

- opencv/opencv — use instead when you need computer vision (detection, tracking, geometric CV), not just image I/O.
- libvips/libvips (pyvips) — use for very large images or high-throughput resize pipelines with low memory.
- imageio/imageio — use when you want a uniform read/write API across scientific, animated, and video formats (it often wraps Pillow/ffmpeg).
- scikit-image/scikit-image — use for image analysis and algorithms operating directly on NumPy arrays.
- uploadcare/pillow-simd — use as a drop-in replacement when Pillow's resize/filter speed is the bottleneck on x86.

## History

| Version | Date | Notes |
|---------|------|-------|
| PIL 1.1.7 | 2009 | Last release of the original PIL by Fredrik Lundh[^1]. |
| Pillow 1.0 | 2010 | Fork begins; focus on installability and packaging[^1]. |
| Pillow 2.0 | 2013 | Python 3 support. |
| Pillow 6.0 | 2019 | Last series supporting Python 2.7. |
| Pillow 7.0 | 2020 | Python 2 dropped; Python 3 only. |
| Pillow 9.0 | 2022 | Continued deprecations of legacy constants/APIs. |
| Pillow 10.0 | 2023 | Removed `Image.ANTIALIAS`, `textsize`, and other long-deprecated APIs; dropped Python 3.7[^2]. |
| Pillow 11.0 | 2024 | Dropped Python 3.8, added newer CPython support. |

## References

[^1]: Pillow documentation, "About" / project history — the fork's origin and relationship to PIL. https://pillow.readthedocs.io/en/stable/about.html
[^2]: Pillow release notes, deprecations and removals (9.x → 10.0). https://pillow.readthedocs.io/en/stable/releasenotes/index.html
[^3]: Pillow docs, `Image.MAX_IMAGE_PIXELS` and decompression-bomb handling. https://pillow.readthedocs.io/en/stable/reference/Image.html
[^4]: Google OSS-Fuzz, continuous fuzzing of Pillow. https://issues.oss-fuzz.com/issues?q=title:pillow

## Tags

python, image-processing, imaging, raster, pil, c-extension, file-formats, thumbnails, computer-graphics, library
