# klauspost/compress

> Pure-Go, drop-in-compatible reimplementations of the standard compression formats (deflate/gzip/zlib/zip, snappy, zstandard), tuned for speed and ratio.

[GitHub repo](https://github.com/klauspost/compress) ·
[License: BSD-3-Clause](https://github.com/klauspost/compress/blob/master/LICENSE)

## Overview

`klauspost/compress` is a collection of Go compression packages maintained by Klaus Post, first published in 2015 as an optimized replacement for the standard library's `compress/flate`[^1]. It has since grown into the de-facto pure-Go home for several formats: `deflate`/`gzip`/`zlib`/`zip` (drop-in replacements for the stdlib packages), `zstd` (a full pure-Go zstandard encoder and decoder), `s2` (a Snappy-compatible extension), plus the entropy coders `huff0` and `fse` and the `gzhttp` HTTP middleware. All of it is wire-compatible with the reference formats — output from this library is readable by any conforming decoder, and vice versa.

The defining goal is "faster and/or smaller than the standard library, with no behavioral surprises." Because the packages mirror stdlib APIs (`gzip.NewWriter`, `flate.NewWriter`, etc.), most projects adopt it by swapping an import path. The library leans on hand-written amd64/arm64 assembly and the `unsafe` package for hot paths, which is the central tradeoff: real throughput gains in exchange for a larger trusted surface than the stdlib. Build tags (`noasm`, `nounsafe`) exist precisely because that tradeoff is not acceptable everywhere.

It is one of the most transitively-vendored Go modules in existence — pulled in by container tooling, object stores, databases and CLI tools wherever zstd or fast gzip is needed. Klaus Post is a MinIO engineer, and the object store is a heavy first-party consumer[^2].

## Getting Started

```bash
go get github.com/klauspost/compress@latest
```

```go
package main

import (
	"bytes"
	"io"

	"github.com/klauspost/compress/zstd"
)

func main() {
	// Encoder/Decoder are expensive to build — create once, reuse.
	enc, _ := zstd.NewWriter(nil, zstd.WithEncoderLevel(zstd.SpeedDefault))
	// EncodeAll is safe for concurrent use across goroutines.
	compressed := enc.EncodeAll([]byte("hello, hello, hello"), nil)
	enc.Close()

	dec, _ := zstd.NewReader(nil, zstd.WithDecoderMaxMemory(64<<20))
	defer dec.Close()
	out, _ := dec.DecodeAll(compressed, nil)
	_ = out

	// Streaming: gzip drop-in — identical API to compress/gzip.
	// gw := gzip.NewWriter(w); io.Copy(gw, r); gw.Close()
	_ = io.Discard
	_ = bytes.NewReader
}
```

## Architecture / How It Works

The repo is a monorepo of independent packages rather than one library. Key ones:

- **`flate` / `gzip` / `zlib` / `zip`** — deflate (RFC 1951[^3]) and its container formats. API-identical to the stdlib, so they slot in by import swap. Adds compression levels and modes the stdlib lacks: `HuffmanOnly`, a `ConstantOutput` fast path, and a *stateless* deflate mode that compresses without a reusable allocation, aimed at highly-concurrent workloads.
- **`zstd`** — a from-scratch pure-Go zstandard (RFC 8878[^4]) codec, not a cgo binding. Encoder levels are `SpeedFastest`, `SpeedDefault`, `SpeedBetterCompression`, and `SpeedBestCompression`. Supports streaming (`NewWriter`/`NewReader`) and one-shot (`EncodeAll`/`DecodeAll`), dictionaries, and a dictionary builder. It does not attempt to match the C reference library's exact ratios at every level, but interoperates fully.
- **`s2`** — an extension of Google's Snappy format. It decodes ordinary Snappy and produces a superset stream with better ratio, concurrent block encode/decode, seekable indexes, and dictionaries. Not a byte-for-byte Snappy replacement in output, but backward-compatible on read.
- **`huff0` / `fse`** — Huffman and Finite State Entropy coders. These are the entropy layer inside `zstd`, exposed as standalone packages for raw use.
- **`gzhttp`** — client/server HTTP middleware for transparent gzip and zstd, including optional BREACH mitigation.

Performance comes from generated/hand-written assembly (match-length search, huff0/fse decode loops) and `unsafe` little-endian loads[^5]. Two build tags gate this: `noasm` forces pure-Go fallbacks (needed on architectures without assembly, or for auditability), and `nounsafe` removes all `unsafe` usage. Both cost throughput. The project supports the current Go release plus the two prior.

## Production Notes

- **Decompressing untrusted input needs explicit limits.** A zstd or deflate stream from the network is a classic decompression-bomb and crash vector. Use `zstd.WithDecoderMaxMemory`, `WithDecoderMaxWindow`, and `WithDecodeAllCapLimit` to bound output. The history is instructive: a gzip stack-exhaustion fix (v1.15.8), multiple zstd "crash on invalid input" hardening fixes (v1.15.9), and a downstream fix for CVE-2025-61728 shipped in v1.18.3[^6]. Treat the decoders as an attack surface and cap them.
- **Reuse Encoders and Decoders.** They allocate substantial buffers (window, tables) and are meant to be long-lived. `EncodeAll` and `DecodeAll` on a single Encoder/Decoder are safe for concurrent use by multiple goroutines; the streaming `Write`/`Read` paths are *not* — one stream per goroutine.
- **Streaming zstd holds memory proportional to window and concurrency.** `WithEncoderConcurrency` / `WithDecoderConcurrency` trade RAM and goroutines for throughput; the defaults scale with `GOMAXPROCS`, which can surprise you in memory-constrained containers.
- **A release was retracted.** v1.18.1 (Oct 2025) is marked RETRACTED in the changelog[^7]; Go's module retraction means `@latest` skips it, but pinned builds should move off it.
- **Assembly limits portability.** On targets without amd64/arm64 asm (some wasm, gccgo, exotic arches) you need `-tags noasm`, and pure-Go performance is materially lower. Security-sensitive builds that forbid `unsafe` want `-tags nounsafe`.
- **It is a deep transitive dependency.** Because so many modules vendor it, version skew across a build graph is common; `go mod graph` will usually show it pulled by several parents at once.

## When to Use / When Not

**Use when:**
- You want faster/smaller gzip, zlib, or zip with no code changes beyond the import path.
- You need pure-Go zstandard or Snappy with no cgo (cross-compilation, static binaries, CGO_ENABLED=0).
- You need zstd/s2 features the stdlib and cgo bindings lack: dictionaries, seekable streams, concurrent block codecs.

**Avoid when:**
- You must forbid `unsafe` and assembly *and* also need peak speed — the fallbacks work but give up the main advantage.
- You need the C reference library's exact output or its absolute top-end ratio at max levels — a cgo binding to libzstd is closer.
- You have zero third-party-dependency constraints and modest performance needs — stdlib `compress/*` is sufficient.

## Alternatives

- google/snappy (Go: golang/snappy) — use when you need canonical Snappy output and none of S2's extensions.
- pierrec/lz4 — use when you specifically need the LZ4 frame/block format in pure Go.
- DataDog/zstd or valyala/gozstd — cgo bindings to libzstd; use when you need the reference library's exact behavior/ratio and can accept cgo.
- facebook/zstd — the upstream C implementation; use when Go is not the constraint.
- golang stdlib compress/gzip · compress/flate — use when you want zero external dependencies and don't need the speed or ratio gains.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2015-07-21 | Repo created as an optimized deflate/gzip drop-in[^1]. |
| — | c. 2019 | Pure-Go `zstd` encoder/decoder added[^8]. |
| 1.16.0 | 2023-02-26 | S2 dictionary support, LZ4 block converter[^7]. |
| 1.17.0 | 2023-09-19 | Experimental zstd dictionary builder, flate limited-window[^7]. |
| 1.18.0 | 2025-02-19 | Unsafe little-endian loaders, flate simplifications[^5]. |
| 1.18.1 | 2025-10-20 | RETRACTED release[^7]. |
| 1.18.3 | 2026-01-16 | Downstream fix for CVE-2025-61728[^6]. |
| 1.18.4 | 2026-02-09 | gzhttp zstd server wrapper, zstd `ResetWithOptions`[^7]. |

## References

[^1]: klauspost/compress README — package overview and usage. https://github.com/klauspost/compress#compress
[^2]: MinIO uses klauspost/compress for object compression; Klaus Post is a MinIO engineer. https://github.com/minio/minio
[^3]: RFC 1951 — DEFLATE Compressed Data Format Specification. https://www.rfc-editor.org/rfc/rfc1951
[^4]: RFC 8878 — Zstandard Compression and the application/zstd Media Type. https://www.rfc-editor.org/rfc/rfc8878
[^5]: Release v1.18.0 — "Add unsafe little endian loaders" (PR #1036). https://github.com/klauspost/compress/releases/tag/v1.18.0
[^6]: Release v1.18.3 — downstream fix for CVE-2025-61728 (golang/go#77102). https://github.com/klauspost/compress/releases/tag/v1.18.3
[^7]: klauspost/compress changelog (releases). https://github.com/klauspost/compress/releases
[^8]: `zstd` subpackage documentation. https://pkg.go.dev/github.com/klauspost/compress/zstd

## Tags

go, golang, compression, decompression, zstd, zstandard, gzip, deflate, snappy, s2, entropy-coding, drop-in-replacement
