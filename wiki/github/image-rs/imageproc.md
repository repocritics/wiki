# image-rs/imageproc

> Pure-Rust image processing primitives — filters, edge detection, drawing, and geometry — built on top of the `image` crate.

[GitHub repo](https://github.com/image-rs/imageproc) ·
[API docs](https://docs.rs/imageproc) ·
[License: MIT](https://github.com/image-rs/imageproc/blob/main/LICENSE)

## Overview

`imageproc` is the algorithms layer of the image-rs ecosystem. The base
[`image`](https://github.com/image-rs/image) crate handles decoding, encoding,
resizing, and pixel storage; `imageproc` adds the operations you actually
compute with — convolutions, gradients, edge and corner detectors, Hough
transforms, morphology, connected components, geometric warps, template
matching, contour tracing, and 2D drawing. It has existed since 2015[^1] and is
the closest thing Rust has to a native, dependency-light "OpenCV-lite" for
classical (non-neural) computer vision and raster graphics work.

Its defining tradeoff is stated plainly in its own README: it explicitly trades
genericity for a consistent, well-tested API[^2]. It is not trying to be maximally
generic over pixel storages, formats, or higher-dimensional tensors, and it is
not GPU-accelerated. Everything runs on the CPU over `image`'s in-memory buffers.
This makes it easy to read, easy to link (no C/C++ toolchain, unlike OpenCV
bindings), and easy to reason about — at the cost of raw throughput and breadth
versus a mature C++ CV stack.

The second thing to know up front: it is still a pre-1.0 crate. The API is
usable and widely depended upon, but it moves in lockstep with `image`'s own
breaking releases, and a bump to a new `image` major version is the single most
common source of churn for downstream users.

## Getting Started

```toml
# Cargo.toml
[dependencies]
image = "0.25"
imageproc = "0.25"
```

```rust
use image::{GrayImage, Luma};
use imageproc::edges::canny;
use imageproc::gradients::sobel_gradients;

fn main() {
    // Load and convert to 8-bit grayscale.
    let img = image::open("input.png").unwrap().to_luma8();

    // Canny edge detection with hysteresis thresholds.
    let edges: GrayImage = canny(&img, 50.0, 100.0);
    edges.save("edges.png").unwrap();

    // Sobel gradient magnitude (returns a 16-bit image).
    let grad = sobel_gradients(&img);
    let _first: Luma<u16> = *grad.get_pixel(0, 0);
}
```

Most functions take an `&image::ImageBuffer` (often `GrayImage` /
`Luma<u8>` for CV work) and return a new buffer — the library favors owned
outputs over in-place mutation, with a few explicit `_mut` drawing variants.

## Architecture / How It Works

`imageproc` is a flat collection of modules over `image`'s `ImageBuffer<P, Vec<S>>`
type. There is no central pipeline object or graph; you call free functions and
thread the buffers yourself. Modules map to CV subfields: `filter`, `gradients`,
`edges`, `corners`, `hough`, `contours`, `distance_transform`, `morphology`,
`region_labelling` (connected components), `template_matching`, `geometric_transformations`,
`integral_image`, `map`, `stats`, and `drawing`.

Two design decisions shape everything downstream:

- **Linear color space assumption.** Functions assume pixels are stored in a
  linear space (like linear RGB), not a gamma-encoded one like sRGB[^3]. Blurring
  or averaging sRGB values directly produces the classic too-dark artifacts. The
  library does not gamma-correct for you; you are responsible for linearizing
  input and re-encoding output if your source is sRGB.
- **Numeric types are explicit and lossy by design.** Gradient and filter
  outputs frequently widen to `i16` / `u16` / `f32` to avoid clipping, and it is
  on the caller to normalize back to `u8` for display or saving. This is a
  frequent source of "my output image is black/white" confusion.

Parallelism is opt-in via [rayon](https://github.com/rayon-rs/rayon): several
functions ship both single-threaded and multi-threaded variants, gated behind
the default `rayon` feature[^2]. The README is candid that the parallel versions
are not always faster — for small images or cheap per-pixel work, the threading
overhead can dominate, and they recommend benchmarking your specific case rather
than assuming the parallel path wins.

## Production Notes

- **Coupled to `image`'s release cadence.** `imageproc`'s public types *are*
  `image` types, so a major `image` bump (e.g. the 0.24 → 0.25 transition) forces
  a coordinated `imageproc` bump and can break your code at both layers at once.
  Pin both crates and upgrade them together.
- **CPU-only, single-node.** There is no GPU backend and no SIMD-guaranteed fast
  path. For large images or real-time video, expect it to be materially slower
  than OpenCV. Profile before committing to it for latency-sensitive pipelines.
- **Feature flags gate real functionality, not just deps.** `text` enables text
  rendering (via a font rasterizer), `fft` enables `phash` and other transform-based
  functions (pulls in `rustfft`), and `display-window` enables `imageproc::window`
  via SDL2 — which requires system SDL2 libraries and will not build in a minimal
  container without them[^2]. Disable defaults (`default-features = false`) if you
  only need core filters and want a lean build.
- **Watch the output bit depth.** Saving a `u16` gradient image straight to PNG,
  or forgetting to normalize a float result, is the most common beginner footgun.
  Use the `map` / normalization helpers before writing to an 8-bit format.
- **API stability.** Pre-1.0 semver means minor versions may rename or reshape
  functions. Vendoring the exact version and reading the changelog before bumping
  is the safe posture for long-lived code.

## When to Use / When Not

**Use when:**
- You want classical CV / raster operations in pure Rust with no C++ toolchain,
  no OpenCV build, and easy cross-compilation.
- Your images fit in memory and CPU throughput is acceptable.
- You are building on the `image` crate already and want operations that compose
  with it natively.

**Avoid when:**
- You need GPU acceleration, real-time video at scale, or the full breadth of
  OpenCV (feature descriptors, calibration, DNN module, video I/O).
- You need higher-dimensional images or maximal genericity over storage/formats —
  explicit non-goals of the project[^2].
- You need a differentiable / tensor-based CV pipeline for ML.

## Alternatives

- twistedfall/opencv-rust — use when you need OpenCV's full algorithm set or GPU; accept the C++ build dependency.
- image-rs/image — use when you only need decode/encode/resize/basic pixel ops, not CV algorithms (imageproc depends on it).
- kornia/kornia-rs — use when you want a tensor-based, ML-adjacent CV toolkit rather than buffer-oriented functions.
- silvia-odwyer/photon — use for WASM/web-targeted photo effects and filters rather than CV primitives.
- rust-cv/cv — use for geometric vision (SLAM, multi-view geometry) where imageproc's 2D scope is too narrow.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-09 | Repository created under the image-rs org[^1]. |
| 0.x | 2016–2023 | Long pre-1.0 line; API grows module by module, tracking `image` 0.2x. |
| 0.24 | 2023 | Tracks `image` 0.24. |
| 0.25 | 2024 | Bumped to `image` 0.25; coordinated ecosystem major bump. |

Still pre-1.0 as of 2026; there has been no 1.0 release. Development remains
active, with commits through mid-2026.

## References

[^1]: GitHub repository metadata, image-rs/imageproc — created 2015-09-27. https://github.com/image-rs/imageproc
[^2]: imageproc README (Goals, Non-goals, Parallelism, Crate Features). https://github.com/image-rs/imageproc/blob/main/README.md
[^3]: imageproc README, "Color Space" section, citing gamma-correction background. https://blog.johnnovak.net/2016/09/21/what-every-coder-should-know-about-gamma/

## Tags

rust, image-processing, computer-vision, edge-detection, convolution, drawing, cpu, crate, mit-license, classical-cv
