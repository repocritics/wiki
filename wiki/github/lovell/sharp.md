# lovell/sharp

> High-performance Node.js image processing — a thin, streaming binding over the libvips C library.

[GitHub repo](https://github.com/lovell/sharp) ·
[Official website](https://sharp.pixelplumbing.com) ·
[License: Apache-2.0](https://github.com/lovell/sharp/blob/main/LICENSE)

## Overview

sharp is a native addon that resizes, converts, and composites images from JavaScript. It was created by Lovell Fuller in 2013 and has become the default image-processing library in the Node ecosystem — it is what Next.js, Astro, Gatsby, and most SSR image-optimization pipelines call under the hood. Its stated advantage is speed: typical resize operations run 4x-5x faster than the fastest ImageMagick/GraphicsMagick settings, achieved by delegating all real work to [libvips](https://github.com/libvips/libvips), a demand-driven, horizontally-threaded C image library.[^1]

The design is deliberately narrow. sharp is not a general graphics toolkit; it is a fluent pipeline builder (`sharp(input).resize(...).webp().toBuffer()`) over the operations libvips exposes: resize, rotate, extract/crop, composite, colour-space and ICC handling, gamma correction, and format encode/decode for JPEG, PNG, WebP, AVIF, GIF, TIFF, and SVG input. Lanczos resampling is the default so quality is not traded for speed.

The defining tension is that sharp is a **native module wrapping a large C library with many codec dependencies**. That is the source of both its performance and nearly all of its operational pain: installation, cross-platform packaging, container builds, and licensing are all more complicated than a pure-JS library. Despite its maturity and ubiquity, sharp has never shipped a 1.0 — it remains on a 0.x line, and packaging has changed under teams' feet more than once.

## Getting Started

```sh
npm install sharp
```

sharp requires a JavaScript runtime with Node-API v9 support: Node.js >= 20.9.0, Deno, or Bun.[^1] Most modern macOS, Windows, and Linux systems need no additional system libraries — prebuilt binaries are downloaded as platform-specific optional dependencies.

```javascript
import sharp from 'sharp';

// Resize and transcode to WebP
await sharp('input.jpg')
  .resize({ width: 320, height: 240, fit: 'cover' })
  .webp({ quality: 80 })
  .toFile('output.webp');

// Streaming: transform a readable stream into a writable one
const pipeline = sharp().rotate().resize(1024).jpeg({ mozjpeg: true });
readableStream.pipe(pipeline).pipe(writableStream);
```

## Architecture / How It Works

sharp itself is a relatively small layer. The fluent API accumulates options on a JavaScript object; nothing executes until an output method (`toFile`, `toBuffer`, or a stream consume) is called. At that point the accumulated pipeline is handed to a C++ binding that constructs and runs a libvips pipeline.

libvips is the part that matters for performance. It is **demand-driven and streaming**: it does not decode the whole image into a bitmap and then transform it. Instead it processes the image in small regions, pulling scanlines through the operation graph on demand, across multiple threads. This is why sharp's memory footprint stays low and roughly constant even for large images, and why it beats ImageMagick, which materializes full buffers. Work runs off the libuv thread pool, so operations are genuinely concurrent and do not block the event loop.

Packaging is the other architecturally significant story. Recent sharp versions ship as a small JS package plus a set of **platform-specific prebuilt binary packages** under the `@img/` npm scope (e.g. `@img/sharp-linux-x64`, `@img/sharp-darwin-arm64`), selected at install time via npm's `optionalDependencies` + `os`/`cpu` fields.[^2] A WebAssembly build (`@img/sharp-wasm32`) exists as a portable fallback. Each prebuilt binary statically links its own copy of libvips and the codec libraries (libwebp, libaom/AVIF, libtiff, libpng, etc.), so no global libvips install is needed on common platforms. This replaced an older model that downloaded a prebuilt libvips during `npm install` (or compiled from source via node-gyp). The change removed the compile step for most users but made **lockfiles and CI matrices the new failure surface**.

## Production Notes

- **Cross-platform installs are the #1 footgun.** Because the correct binary is chosen from `os`/`cpu`, a lockfile generated on macOS ARM can omit the Linux x64 binary your Docker image needs. Symptoms are `Could not load the "sharp" module using the <platform> runtime` at boot. Fixes: install with the target platform flags (`npm install --cpu=x64 --os=linux sharp`), or configure your package manager to include optional deps for all platforms. pnpm and Yarn each have their own escape hatches; monorepos with a single lockfile across dev machines and CI are the classic breakage.
- **Alpine/musl vs glibc.** There are separate `linuxmusl` binaries. A `node:alpine` base image needs the musl variant; mixing them yields load failures. Using a glibc base (`node:slim`, Debian) avoids a whole class of issues.
- **AWS Lambda / serverless.** The bundled binary must match the Lambda architecture (arm64 vs x64). Bundlers that tree-shake or relocate native `.node` files (esbuild, webpack, Next.js standalone output) frequently drop or mispath the binary; mark sharp as external / unbundled.
- **Concurrency and memory.** sharp uses the libuv thread pool (default size 4). CPU-bound resize workloads can saturate it; tune `UV_THREADPOOL_SIZE` and `sharp.concurrency()` to your core count. libvips caches operations and files — under high throughput, call `sharp.cache(false)` and watch the internal cache to avoid unbounded memory and file-descriptor growth.
- **Untrusted input is an attack surface.** sharp decodes many formats through C codecs; malformed images have historically triggered CVEs in the underlying libraries. Set `limitInputPixels`, keep sharp and its prebuilt binaries current (security fixes ship in the codec libs, not sharp's JS), and treat image upload as untrusted parsing.
- **Licensing.** sharp's own code is Apache-2.0, but the prebuilt binaries statically bundle libvips (LGPL-3.0) and various codec libraries under their own terms. For most SaaS/server use this is unremarkable, but redistributing the binaries inside a shipped product warrants an actual license review.

## When to Use / When Not

**Use when:**
- You need fast, memory-efficient resize/transcode on a server (thumbnails, responsive images, on-the-fly optimization).
- You want streaming so you can pipe uploads through transformation without buffering whole files.
- You need correct colour management, ICC profiles, alpha, and modern formats (WebP, AVIF) out of the box.

**Avoid when:**
- You cannot ship or run a native/WASM binary, or you need a dependency-free pure-JS solution — reach for jimp.
- You need broad, exotic image manipulation (drawing, text layout, hundreds of formats, ImageMagick-style scripting) — sharp intentionally does not cover it.
- You want an image-serving product rather than a library — a standalone service handles caching, URLs, and signing that you would otherwise build yourself.

## Alternatives

- imagemagick/imagemagick — use when you need breadth (formats, drawing, effects) over speed; the reference toolkit, but slower and heavier.
- libvips/libvips — use directly when you are not in Node.js; it is the engine sharp binds to, with bindings for Python, Ruby, PHP, Go, and C#.
- jimp-dev/jimp — use when you need pure JavaScript with zero native dependencies and can accept lower performance.
- imgproxy/imgproxy — use when you want a standalone image-processing HTTP service (also libvips-based) instead of an in-process library.
- thumbor/thumbor — use when you want a self-hosted on-the-fly image server with smart cropping and a URL-based API.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2013-11 | Initial release; libvips binding by Lovell Fuller.[^3] |
| 0.x | 2013–2021 | libvips upgrades add WebP, TIFF, then AVIF/HEIF support; migrates native binding to Node-API. |
| 0.33 | 2023-12 | Repackaged as `@img/` platform-specific prebuilt binaries with a WASM fallback; drops install-time libvips download.[^2] |
| 0.34 | 2025 | Ongoing libvips and codec updates; Node.js >= 20.9 / Node-API v9 baseline.[^1] |

sharp remains a pre-1.0 (0.x) project despite being a load-bearing dependency across the JavaScript ecosystem.

## References

[^1]: sharp README and documentation — install requirements, libvips rationale, performance claims. https://sharp.pixelplumbing.com/
[^2]: sharp installation docs — prebuilt `@img/` binaries, cross-platform install, WASM fallback. https://sharp.pixelplumbing.com/install
[^3]: sharp changelog — release history. https://sharp.pixelplumbing.com/changelog

## Tags

javascript, nodejs, image-processing, libvips, native-addon, resize, webp, avif, performance, streaming, thumbnails
