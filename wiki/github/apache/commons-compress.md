# apache/commons-compress

> One Java API over a dozen compression and archive formats — the library almost every JVM tool depends on without knowing it.

[GitHub repo](https://github.com/apache/commons-compress) ·
[Official website](https://commons.apache.org/compress/) ·
[License: Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0)

## Overview

Apache Commons Compress is a Java library that presents a uniform stream-based
API over a wide set of compression codecs (bzip2, gzip, XZ/LZMA, Snappy,
DEFLATE/DEFLATE64, LZ4, Brotli, Zstandard, legacy Unix `.Z`) and archive
containers (ar, cpio, tar, zip, jar, dump, 7z, arj)[^1]. It is a Commons
component — small in surface area, conservative in cadence, and released under
Apache-2.0. The project long predates its GitHub mirror (created 2011); the 1.0
line goes back to 2009[^2].

Its actual importance is as an invisible transitive dependency. Apache Tika,
Ant, Maven plugins, Elasticsearch tooling, Spark, and countless build/packaging
utilities pull it in to read archives without shelling out to `tar`/`unzip`.
That reach is also why its security advisories matter disproportionately: a
decompression-bomb bug here becomes a denial-of-service surface in hundreds of
downstream products.

The defining tension is scope versus depth. Commons Compress aims to read (and
often write) many formats through one factory-driven API, but the formats are
not uniform: some are read-only, several need optional native or third-party
jars at runtime, and the "one stream API" abstraction leaks badly for
random-access formats like zip and 7z.

## Getting Started

```xml
<dependency>
  <groupId>org.apache.commons</groupId>
  <artifactId>commons-compress</artifactId>
  <version>1.28.0</version>
</dependency>
```

```java
// Read a .tar.gz sequentially. GzipCompressorInputStream wraps the raw bytes;
// TarArchiveInputStream iterates entries. Both are AutoCloseable.
import org.apache.commons.compress.archivers.tar.*;
import org.apache.commons.compress.compressors.gzip.*;

try (var fi  = java.nio.file.Files.newInputStream(java.nio.file.Path.of("a.tar.gz"));
     var bi  = new java.io.BufferedInputStream(fi);
     var gz  = new GzipCompressorInputStream(bi);
     var tar = new TarArchiveInputStream(gz)) {
    TarArchiveEntry entry;
    while ((entry = tar.getNextEntry()) != null) {
        if (!tar.canReadEntryData(entry)) continue;   // e.g. unsupported feature
        System.out.println(entry.getName() + " (" + entry.getSize() + " bytes)");
        // tar's InputStream now yields this entry's bytes until the next call
    }
}
```

For format autodetection, `ArchiveStreamFactory` / `CompressorStreamFactory`
sniff magic bytes and return the right stream — convenient, but they can only
detect, not compose (you still stack compressor-over-archive yourself).

## Architecture / How It Works

Two parallel type hierarchies sit under the package root:

- **Compressors** (`compressors.*`) — single-stream codecs.
  `CompressorInputStream` / `CompressorOutputStream` decorate an underlying
  stream. `CompressorStreamFactory` maps names/magic to implementations.
- **Archivers** (`archivers.*`) — multi-entry containers.
  `ArchiveInputStream` exposes `getNextEntry()` returning `ArchiveEntry`
  (name, size, mtime, permissions); `ArchiveOutputStream` takes `putArchiveEntry`
  / `write` / `closeArchiveEntry`.

The clean abstraction is the sequential stream. It breaks for formats that
require random access. Two escape hatches exist:

- **`ZipFile`** — a random-access reader distinct from `ZipArchiveInputStream`.
  It reads the central directory, so it handles entries whose sizes live in a
  trailing data descriptor and gives correct results where the streaming reader
  cannot. Prefer it whenever you have a seekable `File`/`SeekableByteChannel`.
- **`SevenZFile`** — 7z has *no* streaming API at all. It is inherently
  random-access and solid-block; you must supply a seekable source and can pay
  large memory costs decoding solid archives.

Codec coverage is deliberately partial. Several formats are **read-only** (dump,
arj, and — historically — Brotli and DEFLATE64). Several delegate to **optional
third-party jars** that are *not* transitive by default: XZ/LZMA via
`org.tukaani:xz`, Brotli via `org.brotli:dec`, and Zstandard via the native
`com.github.luben:zstd-jni`[^3]. Snappy, LZ4, gzip, bzip2, and DEFLATE are
pure-Java and built in.

## Production Notes

The library's rough edges are almost all about the seams above.

- **Streaming zip is a trap.** `ZipArchiveInputStream` cannot reliably read
  entries that store their sizes in a data descriptor, and silently mis-reports
  in some encrypted or spec-corner cases. If you can seek, use `ZipFile`. The
  streaming reader exists for pipes where you genuinely cannot[^4].
- **Optional dependencies fail at runtime, not compile time.** Ship a fat/shaded
  jar or an Android app that omits `xz`/`zstd-jni` and you get a runtime
  `CompressorException` or `NoClassDefFoundError` only when that format is first
  touched. `zstd-jni` additionally carries platform-specific native binaries —
  a frequent source of "works on my laptop, fails in the container" reports.
- **Decompression bombs.** Because it parses attacker-controlled archives deep
  in many stacks, Commons Compress has shipped a cluster of DoS advisories,
  notably the 2021 series covering crafted 7z/TAR/zip inputs causing excessive
  CPU or memory[^5]. Validate uncompressed sizes and entry counts yourself;
  some readers expose memory-limit knobs (`setMaxMemoryLimitInKb`) but the
  defaults do not cap a malicious archive.
- **Filename encoding.** Non-UTF-8 zips (classic Windows CP437/legacy code
  pages) are a recurring pain; use the `encoding` / Unicode-extra-field options
  on `ZipFile` / `ZipArchiveOutputStream` rather than trusting defaults.
- **Concurrency.** Streams, factories, and readers are not thread-safe; one
  instance per thread. For write throughput, `ParallelScatterZipCreator`
  compresses entries in parallel then merges.
- **Java baseline & deps drift.** Recent 1.x releases target Java 8; the exact
  minimum is pinned in `pom.xml`. Upgrades periodically bump the transitive
  `xz` / `zstd-jni` versions, which can collide with a native `zstd-jni` your
  app pins elsewhere.

## When to Use / When Not

**Use when:**
- You need to read or write more than one archive/compression format through a
  single, stable Java API.
- You must handle formats the JDK lacks entirely (tar, 7z, cpio, ar, XZ, zstd,
  bzip2, LZ4).
- You want pure-Java portability and can accept optional jars for the exotic
  codecs.

**Avoid when:**
- You only need zip/gzip/deflate — `java.util.zip` in the JDK covers that with
  no dependency.
- You need only one high-performance codec (e.g. Zstandard) — bind directly to
  its native library instead of routing through the abstraction.
- You need full 7z *write* support or password-protected multi-format creation —
  native-backed bindings do more than Commons Compress's Java 7z writer.

## Alternatives

- openjdk/jdk (`java.util.zip`) — use when you only need zip, gzip, or raw
  DEFLATE and want zero extra dependencies.
- luben/zstd-jni — use when Zstandard is the only codec you need and you want
  native throughput without the abstraction layer.
- tukaani/xz-java — use when you only need XZ/LZMA; Commons Compress already
  delegates to it, so depend directly if that is all you want.
- lz4/lz4-java — use when LZ4 block/frame speed is the point and you don't need
  containers.
- borisbrodski/sevenzipjbinding — use when you need real 7z write support or
  many formats via native 7-Zip bindings rather than the partial Java 7z writer.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2009 | First Commons release; unified archiver/compressor API[^2]. |
| 1.14 | 2017 | LZ4 (block + framed) and Brotli read support added[^6]. |
| 1.16 | 2018 | Zstandard support via `zstd-jni`[^6]. |
| 1.21 | 2021 | Java 8 baseline; cluster of DoS/security fixes[^5]. |
| 1.26 | 2024 | Dependency and codec maintenance; continued Java 8 target. |
| 1.28.0 | 2026 | Current release line[^1]. |

## References

[^1]: Apache Commons Compress homepage and README — format list and current release. https://commons.apache.org/proper/commons-compress
[^2]: Apache Commons Compress release history (project predates its 2011 GitHub mirror; 1.0 dates to 2009). https://commons.apache.org/proper/commons-compress/changes-report.html
[^3]: Commons Compress "Limitations and Notes" — optional runtime dependencies for XZ, Brotli, and Zstandard. https://commons.apache.org/proper/commons-compress/limitations.html
[^4]: Commons Compress zip documentation — `ZipFile` vs `ZipArchiveInputStream` tradeoffs. https://commons.apache.org/proper/commons-compress/zip.html
[^5]: Apache Commons Compress security advisories (2021 DoS cluster in archive parsers). https://commons.apache.org/proper/commons-compress/security-reports.html
[^6]: Commons Compress changes report — codec additions by version. https://commons.apache.org/proper/commons-compress/changes-report.html

## Tags

java, compression, archive, zip, tar, gzip, bzip2, zstandard, 7z, apache-commons, jvm-library
