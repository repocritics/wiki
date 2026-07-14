# image-rs/image

> Pure-Rust encoding and decoding for common raster image formats, plus a basic set of image processing operations.

[GitHub repo](https://github.com/image-rs/image) ·
[crates.io](https://crates.io/crates/image) ·
[Docs](https://docs.rs/image) ·
[License: Apache-2.0](https://github.com/image-rs/image/blob/main/LICENSE-APACHE)

## Overview

`image` is the de facto standard image library for Rust: when a Rust program needs to open a PNG or JPEG and get pixels out, this is almost always the crate it reaches for[^1]. It provides decoders and encoders for the common raster formats (PNG, JPEG, GIF, BMP, ICO, TIFF, WebP, AVIF, PNM, TGA, DDS, HDR, OpenEXR, QOI, farbfeld), a runtime-polymorphic `DynamicImage` type, an `ImageBuffer<P>` container generic over pixel type, and an `imageops` module of processing routines (resize, blur, crop, rotate, filter). It has existed since 2014 and remains actively maintained by the image-rs organization.

The defining characteristic is that the format codecs are, with few exceptions, written in safe Rust rather than wrapping C libraries like libpng/libjpeg-turbo/giflib. This is `image`'s central tradeoff. The upside is memory safety across the decode path — the historically most CVE-dense part of any image stack — and trivial cross-compilation with no system libraries to link. The downside is that pure-Rust decoders have sometimes lagged the mature C implementations on speed and format-edge-case coverage, and encoding support for some formats (notably AVIF and, historically, WebP) is thinner than the decode side.

The second thing to understand is that `image` is a *codec and container* library first and a *processing* library a distant second. `imageops` covers the basics competently, but anything resembling computer vision, drawing, morphology, or high-throughput resampling lives in sibling or third-party crates. Treating `image` as an ImageMagick replacement is the most common mismatch of expectations.

## Getting Started

```toml
# Cargo.toml — for a library, prefer disabling defaults and opting in
[dependencies]
image = { version = "0.25", default-features = false, features = ["png", "jpeg"] }
```

```rust
use image::{ImageReader, imageops::FilterType};

fn main() -> Result<(), image::ImageError> {
    // Decode: format is guessed from extension, or content with with_guessed_format().
    let img = ImageReader::open("input.jpg")?.decode()?;

    println!("{:?}, {:?}", img.dimensions(), img.color());

    // A few imageops via DynamicImage's convenience methods.
    let out = img
        .resize(256, 256, FilterType::Lanczos3)   // preserves aspect ratio
        .blur(1.5)
        .grayscale();

    out.save("output.png")?; // encoder chosen from extension
    Ok(())
}
```

## Architecture / How It Works

The crate is a thin facade over a family of independent codec crates, most of which live under the image-rs organization and version separately: `png`, `jpeg-decoder`, `gif`, `tiff`, `image-webp`, `ravif`/`avif-decode`, `exr`, `qoi`, and others. `image` itself owns the shared types, the format-dispatch layer, and `imageops`. Because the codecs are separate crates, a decoder bug fix ships as a patch to `png` or `jpeg-decoder` and reaches you through a normal `cargo update` without an `image` release.

Core types:

- **`ImageBuffer<P, Container>`** — width, height, and a flat `Vec<u8>` (or other container) of pixel components, generic over pixel type `P` (`Rgb8`, `Rgba8`, `Luma8`, `LumaA8`, and 16-bit / `f32` variants). This is the concrete, statically-typed image.
- **`DynamicImage`** — an enum over all the supported `ImageBuffer` variants, so the pixel type is resolved at runtime. This is what `decode()`/`open()` return, because the type isn't known until the header is read. Convenience methods delegate to `imageops`. The enum dispatch means per-operation `match` overhead and code bloat, but it is what makes the high-level API ergonomic.
- **`GenericImageView` / `GenericImage`** — the read / read-write traits that `imageops` functions are written against, so they work over `ImageBuffer`, `DynamicImage`, and `SubImage` views uniformly.
- **`ImageDecoder` / `ImageEncoder`** — the per-format traits. `read_image` decodes into a caller-provided byte slice; `dimensions`/`color_type` expose metadata cheaply.

Format selection flows through `ImageReader`, which either trusts the file extension or sniffs magic bytes via `with_guessed_format()`. As of the 0.25 line, additional and third-party decoders can be registered through a plugin interface, with `image-extras`[^2] hosting less-common formats out of the default tree — a move toward keeping the core dependency surface small.

## Production Notes

**Feature flags are load-bearing.** The default feature set pulls in every codec plus `rayon`. For a library or a size- or wasm-sensitive binary, set `default-features = false` and enable only the formats you actually decode[^1]. The README explicitly warns that the default `rayon` multithreading can misbehave on inherently single-threaded targets like `wasm`; disable it there.

**Decompression bombs are a real attack surface.** A small file can declare enormous dimensions and OOM the process on decode. `image` exposes a `Limits` API (max allocation, max dimensions) that you should set before decoding anything untrusted. It is not aggressive by default.

**Debug builds are deceptively slow.** `imageops` routines (resize, blur, convolution) run orders of magnitude slower without optimizations; the README calls this out directly. Benchmark and ship in release mode, and if resampling is a hot path, reach for `fast_image_resize` (SIMD) rather than `imageops::resize` — the built-in Lanczos/Gaussian paths are correct but not the fastest available.

**Encoding is not symmetric with decoding.** AVIF encoding (`ravif`) drags in non-Rust dependencies (`dav1d`, and `nasm` at build time for best performance) via the `avif-native`/`nasm` features; without them you get slower or unavailable paths. WebP encoding has historically been more limited than decoding. Verify the exact encode capabilities of any format you depend on for output, not just input.

**0.x means breaking changes.** The crate is still pre-1.0 after a decade, and minor bumps carry real breakage. The 0.24 → 0.25 transition (2024) reworked feature-flag names, removed deprecated APIs, and consolidated the reader types (`io::Reader` → `ImageReader`). Pin a minor version and read the release notes before upgrading; don't assume `^0.25` is a no-op over `0.24`.

**EXIF orientation is not applied automatically.** Decoded pixels are in stored order; if you need the display-oriented image you must read the orientation tag and rotate/flip yourself. This surprises people handling phone-camera JPEGs.

## When to Use / When Not

**Use when:**
- You need to read or write common raster formats in Rust with no C toolchain and easy cross-compilation.
- Memory safety across the decode path matters (you're parsing untrusted images server-side).
- Your processing needs are basic: resize, crop, rotate, colorspace conversion, format transcoding.
- You want the ecosystem-default that most other Rust image crates interoperate with.

**Avoid (or supplement) when:**
- You need heavy image processing / computer vision — use `imageproc` or bind a dedicated library.
- Resampling throughput is the bottleneck — `fast_image_resize` is substantially faster.
- You need maximum decode speed or exotic format completeness that mature C libraries (or the `zune` crates) provide.
- You need vector/SVG rendering — that's a different problem; `image` is raster-only.

## Alternatives

- image-rs/imageproc — the processing/CV layer (drawing, edges, morphology, geometric transforms) built on top of `image`; use it when `imageops` is too thin.
- Cykooz/fast_image_resize — SIMD-accelerated resizing; use when `imageops::resize` is your hot path.
- etemesi254/zune-image — an alternative pure-Rust codec ecosystem (notably fast JPEG/PNG decoders); use when decode speed dominates.
- libvips/libvips (via bindings) — use when you need format breadth, streaming pipelines, and low memory on very large images, and a C dependency is acceptable.
- RazrFalcon/resvg — use when the input is SVG/vector rather than raster codecs.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2014-05 | First commits; crate published on crates.io. |
| 0.23 | 2020-01 | Long-lived series; broad format coverage matures. |
| 0.24 | 2022-03 | Reworked encoder/decoder traits, animation API changes. |
| 0.25 | 2024-02 | Feature-flag overhaul, `ImageReader` rename, deprecated APIs removed[^3]. |

## References

[^1]: `image` crate README and documentation. https://docs.rs/image / https://github.com/image-rs/image
[^2]: image-extras — out-of-tree and third-party format decoders for `image`. https://github.com/image-rs/image-extras
[^3]: image-rs/image CHANGELOG. https://github.com/image-rs/image/blob/main/CHANGES.md

## Tags

rust, image-processing, image-codec, png, jpeg, encoding, decoding, computer-graphics, library, cross-platform, memory-safety
