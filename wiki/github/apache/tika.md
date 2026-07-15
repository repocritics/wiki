# apache/tika

> A Java toolkit that detects the type of a file and extracts its text and metadata — one API in front of a thousand file formats.

[GitHub repo](https://github.com/apache/tika) ·
[Official website](https://tika.apache.org/) ·
[License: Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0)

## Overview

Apache Tika is a content-detection and extraction library. Given an arbitrary stream of bytes, it identifies the MIME type and then dispatches to the appropriate parser to produce plain text plus a normalized metadata bag[^1]. It advertises support for well over a thousand file types — PDF, the Microsoft Office family, OpenDocument, HTML, XML, RTF, images with EXIF, audio/video containers, archives, mbox/PST mail, and many more. Tika began as a subproject spun out of Apache Lucene (the text-extraction step feeding a search index) and became a top-level Apache project around 2010[^2]. That lineage still shapes it: the primary consumer is a search or ingestion pipeline that needs "just the text."

Tika is best understood as an *aggregator*, not a parser. It writes very little format-specific parsing code itself; instead it wraps established libraries — Apache PDFBox for PDF, Apache POI for the Office formats, jsoup/TagSoup for HTML, Boilerpipe for article extraction, metadata-extractor for images, and dozens more[^3]. Its value is the uniform `Parser` interface, the MIME-detection registry, and the `AutoDetectParser` that ties them together, so callers do not have to know or care which library handles a given byte stream.

The defining tension is exactly that breadth. Every wrapped library is a dependency with its own bugs, CVEs, and pathological inputs, and Tika inherits all of them. A single `tika-parsers-standard-package` pulls in a large transitive dependency tree, and parsing a hostile file can crash, hang, or exhaust memory inside code Tika does not control. Operating Tika safely is largely the discipline of sandboxing and bounding that surface.

## Getting Started

Maven coordinate (the standard parser bundle, via the BOM):

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.apache.tika</groupId>
      <artifactId>tika-bom</artifactId>
      <version>3.x.y</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
<dependencies>
  <dependency>
    <groupId>org.apache.tika</groupId>
    <artifactId>tika-parsers-standard-package</artifactId>
    <type>pom</type>
  </dependency>
</dependencies>
```

The one-liner facade (convenient, but note the caveat below):

```java
import org.apache.tika.Tika;

Tika tika = new Tika();
String text = tika.parseToString(new File("document.pdf"));
```

Command line, no code:

```bash
java -jar tika-app-*.jar --text document.pdf   # extracted text to stdout
java -jar tika-app-*.jar --metadata document.pdf   # metadata only
```

## Architecture / How It Works

Four abstractions carry the whole system:

- **`Detector`** — maps bytes (and optional filename/hint) to a MIME type. Detection uses `tika-mimetypes.xml`: magic-byte signatures, glob patterns, XML root-element checks, and container inspection (e.g. peeking inside a ZIP to distinguish `.docx` from `.xlsx` from a plain archive).
- **`Parser`** — the core SPI. `parse(InputStream, ContentHandler, Metadata, ParseContext)`. Parsers emit text as SAX events into a `ContentHandler` and write key/value pairs into `Metadata`. The SAX design means output is streamed, not necessarily buffered whole.
- **`AutoDetectParser`** — runs detection, then looks up the registered parser for that type via the `ServiceLoader` mechanism (each parser jar declares which MIME types it handles). This is the parser most applications actually use.
- **`Metadata`** — a multi-valued map with an effort at normalization: Dublin Core, XMP, and format-specific keys are mapped toward common `Property` constants, though coverage is uneven across formats.

Output usually flows through a `ContentHandler` decorator. `BodyContentHandler` extracts only the `<body>` of Tika's internal XHTML representation; crucially it takes a write-limit constructor argument that caps extracted characters and throws once exceeded — the standard defense against unbounded output.

For isolation, **`ForkParser`** runs parsing in a separate JVM so a crash or `OutOfMemoryError` in a wrapped library takes down a child process rather than the host. At scale, **tika-pipes** provides an async fetch/parse/emit pipeline for processing large corpora out of process[^4].

Two deployment shapes ship alongside the library: **tika-server** (a JAX-RS REST service — `PUT` a file, get text or metadata back as JSON/plain text) and **tika-app** (the runnable CLI jar). OCR is *not* built in: the `TesseractOCRParser` shells out to an externally installed Tesseract binary, and image-heavy PDFs need it explicitly enabled.

## Production Notes

- **`Tika.parseToString()` silently truncates.** The convenience facade applies a default write limit (historically 100,000 characters) and returns partial text past that. For full extraction use `AutoDetectParser` with a `BodyContentHandler(-1)` and accept the OOM risk you are re-enabling.
- **Untrusted input is a security surface, not an edge case.** Tika has a history of CVEs, most originating in wrapped parsers (PDFBox, POI) or in resource exhaustion — zip bombs, deeply nested containers, catastrophic backtracking. The project's own guidance is to run extraction in an isolated, resource-limited, network-restricted environment; treat `tika-server` as a component you firewall, never expose publicly, and restart on drift[^5].
- **Bound everything.** Set write limits, wall-clock timeouts (via `ForkParser` or pipes), and JVM heap caps. A single malformed file can otherwise pin a core in an infinite loop or allocate until the process dies.
- **Dependency weight.** `tika-parsers-standard-package` drags in a large transitive tree; dependency convergence conflicts with your own PDFBox/POI versions are common. Import `tika-bom` to align module versions, and prefer narrower parser modules if you only handle a few formats.
- **Native/external pieces.** Tesseract OCR and some integration tests depend on external binaries or Docker; those paths are skipped rather than failing the build when the binary is absent, which can mask "OCR isn't actually running" in production.
- **Version/JDK cliffs.** Tika 2.x and its Java 8 support reached End of Life in April 2025[^6]; current lines require newer JDKs (Tika 3.x/4.x target Java 11+/17). The 1.x → 2.x jump was a hard break: parsers were split into modules (`standard`/`standard-package`) and configuration moved to `tika-config` XML, so upgrades are migrations, not version bumps.

## When to Use / When Not

**Use when:**
- You need text/metadata out of many heterogeneous formats behind one JVM API (search indexing, RAG ingestion, e-discovery, DAM).
- You are already on the JVM and want detection + extraction without wiring PDFBox, POI, and a dozen others by hand.
- You need a language-agnostic extraction service — stand up `tika-server` and call it over HTTP from any stack.

**Avoid when:**
- You only ever handle one format — depend on that library directly (PDFBox, POI) and skip the transitive weight.
- You need layout-faithful or structured output (tables, reading order, bounding boxes) for LLM pipelines — Tika's SAX text stream flattens structure; purpose-built converters do better.
- You cannot afford to sandbox untrusted input, or your platform isn't JVM-friendly and the server hop is unacceptable overhead.

## Alternatives

- Unstructured-IO/unstructured — Python-native extraction aimed at LLM/RAG ingestion; use when your stack is Python and you want chunk-ready output over raw text.
- DS4SD/docling — document-to-structured-Markdown conversion with layout/table awareness; use when reading order and tables matter more than format breadth.
- kermitt2/grobid — machine-learning parser specialized for scholarly PDFs (citations, headers); use for academic-paper structure, not general files.
- apache/pdfbox — the PDF engine Tika wraps; use directly when PDF is the only format you touch.
- apache/poi — the Microsoft Office engine Tika wraps; use directly for spreadsheet/document reading with full cell/style access.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2007 | First release as a Lucene subproject[^2]. |
| (TLP) | ~2010 | Graduated to a top-level Apache project. |
| 1.0 | 2011-11 | First stable line; broad format coverage matures. |
| 2.0.0 | 2021-09 | Parser modularization, `tika-config` XML, tika-pipes[^4]. |
| 3.0.0 | 2024 | Newer JDK baseline; further module cleanup. |
| 2.x EOL | 2025-04 | Tika 2.x and Java 8 support end of life[^6]. |

## References

[^1]: Apache Tika project overview. https://tika.apache.org/
[^2]: Apache Tika history and provenance from Lucene. https://tika.apache.org/ (project background); ASF project graduation records.
[^3]: Tika supported formats and underlying parser libraries. https://tika.apache.org/formats.html
[^4]: Apache Tika 2.x/3.x roadmap and tika-pipes. https://cwiki.apache.org/confluence/display/TIKA/Tika+Roadmap+--+2.x%2C+3.x+and+Beyond
[^5]: Apache Tika security guidance. https://tika.apache.org/security.html
[^6]: README note: Tika 2.X and Java 8 reached End of Life, April 2025. https://github.com/apache/tika

## Tags

java, jvm, content-extraction, text-extraction, metadata, mime-detection, document-parsing, pdf, apache, rag-ingestion, ocr
