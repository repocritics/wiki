# apache/commons-io

> Java's oldest general-purpose IO utility library — a grab-bag of stream, file, and filename helpers that predates most of what the JDK now ships natively.

[GitHub repo](https://github.com/apache/commons-io) ·
[Official website](https://commons.apache.org/proper/commons-io) ·
[License: Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0)

## Overview

Apache Commons IO is a collection of utility classes for working with streams, readers/writers, files, filenames, and file filters in Java[^1]. Its first release was in 2005, when the standard library's IO story was `java.io` with no `try-with-resources`, no `java.nio.file`, and no convenience methods for the everyday task of "read this file into a String". For roughly a decade it was near-mandatory in any nontrivial Java project.

The defining tension in 2026 is that **most of Commons IO's original value has been absorbed into the JDK**. `Files.readString`/`writeString` (Java 11), `Files.copy`, `InputStream.transferTo` (Java 9), and `try-with-resources` (Java 7) cover much of what `FileUtils` and `IOUtils` were written for. Yet the library remains one of the most-depended-upon artifacts on Maven Central, almost entirely as a transitive dependency. Its 1,075 GitHub stars badly understate its footprint — this is infrastructure, not a trend repo; it arrives in your build because something else pulled it in.

What justifies keeping it on new code is the long tail the JDK still doesn't cover cleanly: composable file filters, filename manipulation that is OS-aware without touching the filesystem, tailing/monitoring, byte-order and endian helpers, and a large set of decorator streams. It is maintained conservatively — new features are additive, breaking changes are rare, and the JDK baseline stays low.

## Getting Started

Maven (note the legacy `commons-io` groupId, not `org.apache.commons`):

```xml
<dependency>
  <groupId>commons-io</groupId>
  <artifactId>commons-io</artifactId>
  <version>2.22.0</version>
</dependency>
```

```java
import org.apache.commons.io.FileUtils;
import org.apache.commons.io.FilenameUtils;
import java.io.File;
import java.nio.charset.StandardCharsets;

// Read/write whole files (JDK Files.readString covers this on Java 11+)
String text = FileUtils.readFileToString(new File("in.txt"), StandardCharsets.UTF_8);
FileUtils.writeStringToFile(new File("out.txt"), text, StandardCharsets.UTF_8);

// Filesystem-free filename logic — the part the JDK still doesn't do well
String ext  = FilenameUtils.getExtension("/var/data/report.csv"); // "csv"
String base = FilenameUtils.getBaseName("/var/data/report.csv");  // "report"
```

## Architecture / How It Works

Commons IO is a flat set of packages, not a framework — there is no core object graph, most classes are stateless static-method holders or thin stream decorators:

- **`org.apache.commons.io`** — the top-level utility classes: `IOUtils` (copy/read/close on streams), `FileUtils` (whole-file operations, directory copy/delete), `FilenameUtils` (string-only path manipulation), `FileSystemUtils`, `ByteOrderMark`, `EndianUtils`.
- **`.input` / `.output`** — decorator `InputStream`/`OutputStream`/`Reader`/`Writer` implementations: `TeeInputStream`, `CountingInputStream`, `BoundedInputStream`, `NullOutputStream`, `ByteArrayOutputStream` (a resizable variant), `Tailer` for `tail -f`-style following.
- **`.filefilter`** — composable `IOFileFilter` predicates (`AndFileFilter`, `SuffixFileFilter`, `AgeFileFilter`, `WildcardFileFilter`) usable with both legacy `File` and NIO `Path` walks.
- **`.comparator`** — `File` comparators (name, size, last-modified) for sorting directory listings.
- **`.monitor`** — `FileAlterationObserver`/`FileAlterationMonitor`, a polling-based filesystem watcher that predates and differs from `java.nio.file.WatchService`.
- **`.file`** — added in the 2.7 line: `PathUtils` and NIO2-based (`java.nio.file.Path`) equivalents of the older `File` helpers, plus `DeletingPathVisitor`/`CopyDirectoryVisitor`[^2].
- **`.function`** — added in the 2.12 line: `IOSupplier`, `IOFunction`, `IOStream`, and `Uncheck`, wrappers that let `IOException`-throwing lambdas flow through the `java.util.function` world[^3].

The library ships a JPMS `Automatic-Module-Name` of `org.apache.commons.io` but is not a full module. There is deliberately no coupling between packages: you can use `FilenameUtils` without ever touching the filesystem.

## Production Notes

**The groupId footgun.** Commons IO kept the pre-2.x Maven coordinates `commons-io:commons-io`, unlike siblings that moved to `org.apache.commons` (e.g. `org.apache.commons:commons-lang3`). Old build files and copy-pasted snippets sometimes reference `org.apache.commons:commons-io`, which resolves to stale artifacts. Always use groupId `commons-io`.

**CVE-2021-29425.** In versions before 2.7, `FileNameUtils.normalize` could be coaxed into limited path traversal (a leading `//../` style input escaping the intended directory)[^4]. If you pin an old version transitively, force-upgrade to ≥ 2.7. Security scanners flag this frequently because ancient commons-io is a common transitive straggler.

**Prefer the JDK for the basics.** On Java 11+, `Files.readString`, `Files.writeString`, `Files.copy`, and `InputStream.transferTo` are standard-library, dependency-free, and often faster. Reaching for `FileUtils.readFileToString` on new code mostly adds a jar for no gain. Keep Commons IO for the parts with no JDK equivalent (filters, `Tailer`, `FilenameUtils`, endian/BOM handling, monitoring).

**`IOUtils.closeQuietly` is deprecated.** It was the right tool before `try-with-resources` (Java 7); the swallow-all-exceptions pattern is now discouraged and the method is deprecated. Use `try-with-resources`.

**`FileUtils.deleteDirectory` and symlinks.** Directory deletion/copy helpers have historically been sensitive to symlinks and concurrent modification; on large or actively-written trees prefer the NIO2 `PathUtils` variants in the `.file` package, which give more explicit control over link handling.

**JDK baseline.** The 2.x line has stayed on a low JDK floor (Java 8 for most of its history) — check the `maven.compiler.source` property in the release's `pom.xml` for the exact minimum, as the README directs. This conservatism is a feature: upgrades rarely force a language-level bump.

## When to Use / When Not

**Use when:**
- You need composable file filters, filename parsing, `Tailer`, endian/BOM handling, or polling-based file monitoring — the parts with no clean JDK equivalent.
- You are on an older JDK (Java 8) and want the ergonomic whole-file/stream helpers.
- It is already a transitive dependency and you just want to use what is on the classpath.

**Avoid when:**
- You are on Java 11+ and only need read-file / write-file / copy-stream basics — the JDK now covers these without a dependency.
- You need archive or compression formats (zip/tar/7z/gzip streaming) — that is `commons-compress`, not this.
- You want a modern Kotlin/Android-first buffered IO model — Okio fits better.

## Alternatives

- google/guava — `com.google.common.io` (`ByteStreams`, `CharStreams`, `Files`) overlaps heavily; use it when you already depend on Guava rather than adding a second utility jar.
- openjdk `java.nio.file` (the JDK itself) — use `Files.readString`/`writeString`/`copy` and `InputStream.transferTo` for the common cases on Java 11+ with zero dependencies.
- apache/commons-compress — use when you actually need archive/compression formats, which Commons IO deliberately does not handle.
- square/okio — use for Kotlin/Android buffered-IO ergonomics and a `Source`/`Sink` model instead of `java.io` decorators.
- apache/commons-lang — sibling utility library for `String`/reflection/`Object` helpers; commonly paired with, but orthogonal to, Commons IO.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2005 | First release; IO utilities split out of the Jakarta Commons ecosystem. |
| 2.0 | 2010 | Generics, expanded `filefilter`/`comparator` packages, API cleanup. |
| 2.5 | 2016 | Broader filter/observer surface; steady additive growth. |
| 2.6 | 2017 | `closeQuietly` deprecated in favor of try-with-resources. |
| 2.7 | 2020 | NIO2 `org.apache.commons.io.file` (`PathUtils`); fix for CVE-2021-29425[^4]. |
| 2.12 | 2023 | `org.apache.commons.io.function` (`IOSupplier`/`IOStream`/`Uncheck`)[^3]. |
| 2.22.0 | 2026 | Current release per project README[^1]. |

## References

[^1]: Apache Commons IO homepage and README. https://commons.apache.org/proper/commons-io
[^2]: Apache Commons IO Javadoc, package `org.apache.commons.io.file`. https://commons.apache.org/proper/commons-io/apidocs/org/apache/commons/io/file/package-summary.html
[^3]: Apache Commons IO Javadoc, package `org.apache.commons.io.function`. https://commons.apache.org/proper/commons-io/apidocs/org/apache/commons/io/function/package-summary.html
[^4]: CVE-2021-29425 — limited path traversal in `FileNameUtils.normalize` before Commons IO 2.7. https://nvd.nist.gov/vuln/detail/CVE-2021-29425

## Tags

java, jvm, io, file-utilities, streams, apache-commons, standard-library, filesystem, utility-library, maven
