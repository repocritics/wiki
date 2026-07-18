# rust-lang/flate2-rs

> Rust's de facto standard DEFLATE/zlib/gzip library — a thin streaming API
> over swappable compression backends, pure-Rust by default.

[GitHub repo](https://github.com/rust-lang/flate2-rs) ·
[Documentation](https://docs.rs/flate2) ·
License: MIT OR Apache-2.0

## Overview

`flate2` provides streaming compression and decompression for the three
DEFLATE-based formats — raw deflate, zlib, and gzip — behind Rust's standard
`Read`/`Write`/`BufRead` traits[^1]. Started by Alex Crichton in 2014 as
bindings to the miniz C library, it was later transferred to the rust-lang
organization, where it is maintained today alongside `libz-sys`.

The star count (~1.1k) wildly understates its position: `flate2` is one of the
most-downloaded crates on crates.io[^2], sitting in the dependency graph of
cargo itself, `tar`, `reqwest`, `zip`, and most of the HTTP and archive
ecosystem. It is infrastructure, not an application library — almost nobody
stars it, almost everybody compiles it.

Its defining design decision is the **pluggable backend**: the crate contains
very little codec logic of its own. By default it delegates to `miniz_oxide`
(a safe, pure-Rust port of miniz); cargo features swap in `zlib-rs` (unsafe
but fast pure Rust), stock zlib, or zlib-ng (C)[^1]. This gives one stable API
over a performance/portability/supply-chain tradeoff you pick at build time —
and, as a consequence, a set of cargo feature-unification footguns you inherit
(see Production Notes).

## Getting Started

```toml
# Cargo.toml
[dependencies]
flate2 = "1"
```

```rust
use std::io::prelude::*;
use flate2::Compression;
use flate2::write::GzEncoder;
use flate2::read::GzDecoder;

fn main() -> std::io::Result<()> {
    // Compress: wrap any Write sink
    let mut enc = GzEncoder::new(Vec::new(), Compression::default());
    enc.write_all(b"hello world")?;
    let compressed = enc.finish()?; // finish() is mandatory — see Production Notes

    // Decompress: wrap any Read source
    let mut dec = GzDecoder::new(&compressed[..]);
    let mut out = String::new();
    dec.read_to_string(&mut out)?;
    assert_eq!(out, "hello world");
    Ok(())
}
```

Compression levels run 0–9 (`Compression::none()`, `fast()`, `best()`);
`Compression::default()` is level 6, matching zlib's default.

## Architecture / How It Works

The crate is three layers of plumbing over a codec it mostly does not own:

1. **I/O adapters** — the public surface. Each format has encoder/decoder
   types in three flavors: `flate2::read::*` (wrap a `Read`, pull-based),
   `flate2::write::*` (wrap a `Write`, push-based), and `flate2::bufread::*`
   (wrap a `BufRead`, avoids an internal buffer copy — the fastest option when
   you already have buffered input). The nine-way matrix (3 formats × 3 trait
   flavors) is verbose but mechanical.
2. **Raw streaming core** — `Compress` and `Decompress` expose a
   zlib-`inflate`/`deflate`-style buffer-in/buffer-out API with `FlushCompress`
   / `FlushDecompress` control, for callers doing their own buffer management
   (HTTP servers, protocol implementations).
3. **Backend FFI shim** — cargo features select exactly one codec:
   `miniz_oxide` (default; 100% safe Rust), `zlib-rs` (pure Rust with
   `unsafe`, fastest overall per Trifecta Tech's benchmarks[^3]), `zlib`
   (stock C zlib via `libz-sys`), `zlib-ng` (C, SIMD-optimized), or
   `zlib-ng-compat` (zlib-ng in zlib-compat mode, shareable with C
   dependencies)[^1].

Gzip-specific logic (header parsing, filename/mtime fields, CRC32 via
`crc32fast`) lives in `flate2` itself; the backends only see raw deflate
streams. Non-default backends require `default-features = false`, because
cargo features are additive and the backends are mutually exclusive.

## Production Notes

- **`GzDecoder` stops at the first gzip member.** The gzip format allows
  concatenated members, and files produced by `pigz`, `bgzip`, or naive
  `cat a.gz b.gz` contain several. `GzDecoder` reads member one and reports
  EOF — silent truncation, not an error. Use `MultiGzDecoder` when input
  provenance is not fully under your control[^4]. This is the single most
  common flate2 bug report in downstream projects.
- **Call `finish()` on write-side encoders.** The gzip/zlib trailer (CRC +
  length) is emitted by `finish()`. The `Drop` impl attempts a final flush but
  swallows any I/O error, so relying on drop can produce silently corrupt
  output on a full disk or closed pipe.
- **Backend selection is a whole-graph negotiation.** With `zlib-ng-compat`,
  if any crate in your dependency graph requests stock zlib (or uses
  `libz-sys` without `default-features = false`), you silently get stock zlib
  instead of zlib-ng[^5]. The `zlib-ng` feature avoids this by linking zlib-ng
  independently. Audit with `cargo tree -i libz-sys` after changing features.
- **Throughput.** The safe default (`miniz_oxide`) is the slowest of the
  backends. For decompression-heavy services, switching to `zlib-rs` or
  `zlib-ng` is a one-line `Cargo.toml` change worth benchmarking; Trifecta
  Tech measured zlib-rs beating all C implementations overall, with zlib-ng
  ahead in some cases[^3].
- **No decompression-bomb protection.** A tiny deflate stream can expand to
  gigabytes. Cap output yourself (`Read::take` on the decoder, or check
  `Decompress::total_out` in the raw API) before feeding untrusted input.
- **Single-threaded.** One stream, one core — there is no `pigz`-style
  parallel compression. Parallelize at the file/chunk level yourself if you
  need it.
- **MSRV policy is "stable minus one", loosely.** The crate targets the
  current and previous stable compiler, and the `Cargo.toml` `rust-version`
  may be bumped in minor releases[^1]. Conservative-toolchain environments
  should pin a minor version.

Maintenance is healthy: the repo lives under the rust-lang org, has 29 open
issues against an enormous user base, and last saw pushes in June 2026.

## When to Use / When Not

**Use when:**
- You need gzip/zlib/deflate — HTTP content encoding, `.tar.gz`, PNG-adjacent
  tooling, git-style object storage. This is the ecosystem-default choice.
- You want a safe pure-Rust default with the option to buy speed later by
  flipping a backend feature, without touching call sites.
- You need streaming compression over arbitrary `Read`/`Write` pipes.

**Avoid when:**
- You control both ends of the wire: Zstandard (better ratio and speed than
  DEFLATE at almost every operating point) or LZ4 (raw speed) are better
  formats; flate2 only makes sense where DEFLATE is imposed on you.
- You are in async code: flate2's adapters are blocking `std::io`; wrapping
  them in `spawn_blocking` works but `async-compression` is the idiomatic path.
- You need `no_std`: use `miniz_oxide` directly, which supports it.

## Alternatives

- Frommi/miniz_oxide — use directly when you want the pure-Rust DEFLATE codec
  without flate2's wrapper layer, or need `no_std`.
- trifectatechfoundation/zlib-rs — the fast pure-Rust zlib; use directly (or
  via its `libz-rs-sys` C API) when replacing zlib outside flate2's API.
- rust-lang/libz-sys — raw zlib/zlib-ng FFI bindings; use when you need the
  exact C zlib API surface for interop.
- Nullus157/async-compression — use for tokio/futures `AsyncRead`/`AsyncWrite`
  compression adapters (gzip, zstd, brotli and more) instead of blocking I/O.
- gyscos/zstd-rs — use when you control the format choice and want Zstandard's
  ratio/speed instead of DEFLATE.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2014-07 | Initial release by Alex Crichton; C miniz bindings. |
| 1.0 | 2018 | API-stable 1.0; the crate has stayed on 1.x since. |
| 1.0.x (2020) | 2020 | Pure-Rust `miniz_oxide` became the default backend, dropping C from the default build[^6]. |
| 1.1.0 | 2025-02 | `zlib-rs` backend support added[^6]. |

## References

[^1]: flate2-rs README. https://github.com/rust-lang/flate2-rs/blob/main/README.md
[^2]: crates.io — flate2 (all-time download counts). https://crates.io/crates/flate2
[^3]: Trifecta Tech Foundation, "zlib-rs is faster than C". https://trifectatech.org/blog/zlib-rs-is-faster-than-c/
[^4]: docs.rs — `flate2::read::MultiGzDecoder` (vs `GzDecoder` single-member behavior). https://docs.rs/flate2/latest/flate2/read/struct.MultiGzDecoder.html
[^5]: libz-sys README — zlib-ng-compat feature-unification caveat. https://github.com/rust-lang/libz-sys/blob/main/README.md
[^6]: flate2-rs release history. https://github.com/rust-lang/flate2-rs/releases

## Tags

rust, compression, deflate, gzip, zlib, zlib-ng, streaming, io, systems, library
