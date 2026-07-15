# square/okio

> A segmented-buffer I/O library that complements `java.io`/`java.nio`, extracted from OkHttp and now Kotlin Multiplatform.

[GitHub repo](https://github.com/square/okio) ·
[Official website](https://square.github.io/okio/) ·
[License: Apache-2.0](https://github.com/square/okio/blob/master/LICENSE.txt)

## Overview

Okio is a small I/O library from Square that began as an internal component of
OkHttp[^1] and was spun out because the buffering primitives were generally
useful beyond an HTTP client. Its premise is that `java.io` (blocking streams,
lots of small copies) and `java.nio` (channels/buffers with a famously
error-prone `flip()`/`position()` model) are both awkward, and that a single
pair of interfaces — `Source` (read) and `Sink` (write) — plus a good buffer
type covers the vast majority of real I/O.

The defining design choice is the segmented `Buffer`: a mutable byte sequence
backed by a linked list of fixed-size segments that can be **moved between
buffers without copying**. This makes operations like "read the framing bytes
off the front, hand the rest to a decoder" cheap, and it is why Okio is the
substrate under OkHttp, Moshi, Wire, and much of the Square/Android networking
stack. It is not a networking or async framework; it is the layer directly above
raw sockets and files.

As of Okio 3, the library is Kotlin-first and multiplatform: the same API
targets the JVM, Android, Kotlin/Native (iOS, macOS, Linux), and JavaScript,
and version 3 added a `FileSystem` abstraction so file access is no longer
JVM-only[^2]. The tradeoff of that reach is that not every capability exists on
every target, and behavior (especially around the filesystem and timeouts)
differs by platform.

## Getting Started

```kotlin
// Gradle (Kotlin DSL)
dependencies {
    implementation("com.squareup.okio:okio:3.9.0")
}
```

```kotlin
import okio.FileSystem
import okio.Path.Companion.toPath
import okio.buffer

fun main() {
    val path = "notes.txt".toPath()
    val fs = FileSystem.SYSTEM

    // Write: sink() -> buffered -> use{} closes and flushes
    fs.sink(path).buffer().use { sink ->
        sink.writeUtf8("hello okio\n")
        sink.writeInt(42)
    }

    // Read: source() -> buffered; readUtf8Line() decodes without manual byte math
    fs.source(path).buffer().use { source ->
        println(source.readUtf8Line())   // "hello okio"
        println(source.readInt())        // 42
    }
}
```

`Buffer` implements both `BufferedSource` and `BufferedSink`, so it doubles as an
in-memory scratch space with the same API you use for files and sockets.

## Architecture / How It Works

Four types carry the whole library:

- **`Source` / `Sink`** — minimal one-method interfaces (`read`/`write` into a
  `Buffer`, plus `Timeout` and `close`). They are the `InputStream`/
  `OutputStream` analogues, but they always transfer through a `Buffer`, so there
  are no single-byte read paths to accidentally use.
- **`BufferedSource` / `BufferedSink`** — the rich API layer added by
  `.buffer()`: `readUtf8`, `readInt`, `readByteString`, `indexOf`, `select`,
  `writeUtf8`, etc. Buffering here is explicit, not hidden inside every stream.
- **`Buffer`** — a mutable byte sequence implemented as a circular linked list of
  `Segment`s. Each segment wraps a fixed-size (8 KiB) byte array with `pos`/
  `limit` cursors. Moving bytes from one buffer to another re-links segments
  instead of copying them, and a `SegmentPool` recycles segments to keep
  allocation and GC pressure low. Segments can also be **shared** copy-on-write,
  which is how `Buffer.snapshot()` and `ByteString` avoid duplicating data.
- **`ByteString`** — an immutable, indexed sequence of bytes with cheap
  `utf8()`, `hex()`, `base64()`, and `md5()`/`sha256()` accessors. It is the
  "immutable String, but for bytes" type used pervasively as map keys and
  protocol tokens.

Around these sit adapters and codecs: `source()`/`sink()` bridges to
`InputStream`, `OutputStream`, `Socket`, and (on JVM) `java.io.File`/`Path`;
`GzipSource`/`GzipSink`, `InflaterSource`/`DeflaterSource`; `HashingSource`/
`HashingSink` for streaming digests; and cipher sources/sinks. `Timeout`
provides a unified timeout + deadline model, with a watchdog thread powering
async socket timeouts.

In Okio 3 the `FileSystem` interface abstracts file access (`read`, `write`,
`list`, `metadata`, `atomicMove`) with platform implementations: NIO on the JVM,
POSIX on Kotlin/Native, and the Node `fs` module on JavaScript[^2]. The
`okio-fakefilesystem` artifact provides an in-memory `FakeFileSystem` for tests.

## Production Notes

- **Explicit buffering is a feature and a footgun.** A raw `Source`/`Sink` from
  `.source(file)` is unbuffered; forgetting `.buffer()` gives correct but
  syscall-heavy I/O. Conversely, wrapping an already-buffered stream double-
  buffers. Buffer once, at the boundary.
- **`BufferedSource`/`BufferedSink` are not thread-safe.** They assume a single
  reader/writer per stream. Sharing one across threads corrupts segment state.
- **EOF semantics differ by method.** `read()` returns `-1` at end of input;
  `readFully`, `readUtf8Line` on a truncated stream, and the fixed-width
  `readInt`/`readLong` throw `EOFException`. Handle the exception path, not just
  the `-1`.
- **Always `close()` / `use {}`.** Buffered sinks hold unflushed bytes in
  segments; skipping close loses the tail of your data. `use {}` flushes then
  closes.
- **Multiplatform gaps.** Code written against the JVM `FileSystem` may hit
  unimplemented operations or different symlink/permission behavior on Native or
  JS. `atomicMove` and metadata fields in particular vary by platform.
- **Kotlin/Native memory.** On older Kotlin/Native memory models, sharing Okio
  objects across workers was constrained; the new memory manager relaxed this,
  but pin your Kotlin and Okio versions together when targeting Native.
- **Segment size is fixed at 8 KiB.** For workloads dominated by many tiny reads
  this is efficient; for streaming very large payloads the segment-pool ceiling
  and per-segment overhead are worth profiling rather than assuming zero-copy
  everywhere.

## When to Use / When Not

**Use when:**
- You are on Kotlin/Android or Kotlin Multiplatform and want one I/O API across
  JVM, Native, and JS.
- You parse or frame binary/text protocols and want cheap buffer hand-off,
  `ByteString` keys, and streaming hashes/compression.
- You already depend on OkHttp, Moshi, or Wire — Okio is already on your
  classpath and is the idiomatic buffer type.

**Avoid when:**
- You are on plain Java with no Kotlin and only need `copy(in, out)` helpers —
  Guava or Commons IO are lower-friction.
- You need memory-mapped files, direct/off-heap buffers, or async NIO channels —
  that is `java.nio` / Netty territory.
- You want the JetBrains-blessed stdlib path — evaluate `kotlinx-io`, which is
  based on Okio's design (see Alternatives).

## Alternatives

- Kotlin/kotlinx-io — JetBrains' multiplatform I/O library, derived from Okio's
  segmented-buffer design; use it when you want the official Kotlin-team library
  and can accept a younger, still-evolving API.
- google/guava — `com.google.common.io` (`ByteStreams`, `CharStreams`,
  `Files`); use when you are already on Guava and need simple JVM copy/stream
  utilities rather than a buffer model.
- apache/commons-io — classic `java.io` helpers (`IOUtils`, `FileUtils`); use on
  plain JVM projects with no Kotlin or multiplatform requirement.
- netty/netty — `ByteBuf` pooled/zero-copy buffers; use when building
  high-throughput async network protocols that need reference-counted off-heap
  buffers, not file/stream ergonomics.
- java.nio (JDK) — use directly when you need memory-mapped files or direct byte
  buffers and want zero third-party dependencies.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2014 | Extracted from OkHttp as a standalone Java library[^1]. |
| 2.0 | 2018-08 | Rewrite in Kotlin, kept largely source-compatible with 1.x. |
| 3.0 | 2021-10 | Kotlin Multiplatform; new `FileSystem`/`Path` API and `FakeFileSystem`[^2]. |

## References

[^1]: Okio README and project site — "It started as a component of OkHttp." https://square.github.io/okio/
[^2]: Okio `FileSystem` documentation — multiplatform file access (JVM/NIO, POSIX, Node). https://square.github.io/okio/file_system/

## Tags

kotlin, java, android, kotlin-multiplatform, io, buffer, bytestring, streaming, filesystem, square, jvm
