# apache/pdfbox

> A Java library for creating, manipulating, extracting from, and rendering PDF documents — the long-standing open-source default on the JVM.

[GitHub repo](https://github.com/apache/pdfbox) ·
[Official website](https://pdfbox.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/pdfbox/blob/trunk/LICENSE.txt)

## Overview

PDFBox is an Apache Software Foundation project for programmatic work with PDF files in Java: creating documents, editing existing ones, extracting text and images, filling forms, splitting/merging, signing, and rendering pages to raster images[^1]. It began as a personal project by Ben Litchfield around 2002, entered the Apache Incubator in 2008, and became a top-level project in 2009. It predates most of the current PDF-tooling ecosystem and shows its age in places, but it remains the permissively-licensed baseline that most JVM PDF work either uses directly or is measured against.

The defining tradeoff is **license versus feature depth**. PDFBox is Apache-2.0 — you can ship it in closed-source commercial software with no reciprocal obligations. Its main rival, iText, moved to AGPL (with a paid commercial license) and carries a richer high-level API for layout, tagging, and PDF/A/PDF/UA generation. Teams pick PDFBox when the licensing matters more than convenience, and accept that many operations (laying out text, drawing tables, precise typography) are lower-level and more manual than in the commercial alternatives.

The second defining reality is that PDFBox is a **parser of adversarial input**. Real-world PDFs are frequently malformed, and the format is a sprawling ISO 32000 specification with decades of quirks. PDFBox is comparatively good at recovering from broken files, but this also means it periodically ships CVEs (DoS via crafted fonts/streams, resource exhaustion) — anyone feeding it untrusted uploads should treat it as an attack surface and keep it patched.

## Getting Started

Maven coordinate (single-artifact dependency pulls in `fontbox`):

```xml
<dependency>
  <groupId>org.apache.pdfbox</groupId>
  <artifactId>pdfbox</artifactId>
  <version>3.0.5</version>
</dependency>
```

Extract text from a PDF (PDFBox 3.x API — `Loader` replaced the old static `PDDocument.load`):

```java
import org.apache.pdfbox.Loader;
import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.text.PDFTextStripper;
import java.io.File;

try (PDDocument doc = Loader.loadPDF(new File("input.pdf"))) {
    PDFTextStripper stripper = new PDFTextStripper();
    stripper.setSortByPosition(true);   // off by default; see limitations
    System.out.println(stripper.getText(doc));
}
```

Render a page to a PNG:

```java
import org.apache.pdfbox.rendering.PDFRenderer;
import javax.imageio.ImageIO;

try (PDDocument doc = Loader.loadPDF(new File("input.pdf"))) {
    var img = new PDFRenderer(doc).renderImageWithDPI(0, 150);  // page 0 at 150 DPI
    ImageIO.write(img, "PNG", new File("page0.png"));
}
```

Building from source needs Java 11+ and Maven 3 (`mvn clean install`)[^2]; the produced 3.x artifacts run on Java 8+.

## Architecture / How It Works

PDFBox is layered over the PDF file format's own object model, called **COS** (Carousel Object System — a nod to Adobe's original codename)[^1]:

- **COS layer** — `COSDocument`, `COSDictionary`, `COSArray`, `COSStream`, `COSName`, `COSString`. This is the raw graph of PDF objects exactly as they appear in the file. `COSParser` reads the cross-reference table (or reconstructs it when broken) and materializes this graph.
- **PD layer** — `PDDocument`, `PDPage`, `PDPageContentStream`, `PDFont`, `PDAcroForm`. A higher-level, typed wrapper over the COS graph that most application code targets.
- **Operations** — `PDFTextStripper` walks content streams and reassembles text; `PDFRenderer` rasterizes pages via Java2D; `PDPageContentStream` writes drawing operators for content creation.

It ships as several Maven modules: `pdfbox` (core), `fontbox` (TrueType/Type1/CFF/OpenType font parsing, used internally and usable standalone), `xmpbox` (XMP metadata), `preflight` (PDF/A-1b validation), `pdfbox-tools` (command-line utilities), and `pdfbox-debugger` (a Swing inspector for the COS tree). Encryption is delegated to the **Bouncy Castle** libraries, which is why encrypted-PDF handling pulls in an extra dependency and triggers export-control notices[^3].

The 3.0 line (2023) was a breaking API cleanup: the `Loader` entry point replaced static loaders, `MemoryUsageSetting` was reworked into the `RandomAccessStreamCache` / `IOStreams` model, the CLI moved to a single `pdfbox-app` executable jar with subcommands, and the long-deprecated `jempbox` (legacy XMP) was removed[^4]. Code written for 2.0 does not compile unchanged against 3.0.

Text extraction is the subsystem that surprises newcomers most. PDF stores text as positioned glyph runs, not logical reading order, and glyph codes may map to a document-internal encoding rather than Unicode. PDFBox does not sort by position unless told to, and when a font embeds a custom encoding without a `ToUnicode` map, extraction returns meaningless glyph indices that only OCR can recover[^5].

## Production Notes

**Memory on large or hostile files.** By default PDFBox buffers document data in memory; large PDFs (hundreds of MB, many images) can OOM. The mitigation is a scratch-file / temp-file backed cache — in 3.x via `Loader.loadPDF(file, ...)` with an `IOStreams`/`RandomAccessStreamCache` that spills to disk. Set this before you discover it in production.

**Untrusted input is a security surface.** PDFBox has a track record of CVEs — infinite loops and stack overflows in font/CMap parsing, resource exhaustion from crafted streams. If you accept user-uploaded PDFs, run extraction in a sandbox or a resource-capped worker, set page/time limits, and keep the dependency current. Do not assume a `try/catch` around `Loader.loadPDF` bounds CPU or memory usage.

**Rendering fidelity is good, not perfect.** `PDFRenderer` covers the common cases well but is not a full Adobe-grade renderer. Expect occasional deviations on complex transparency groups, uncommon blend modes, certain CJK/embedded-font edge cases, and some JBIG2/JPEG2000 images (JPEG2000 needs the optional `jai-imageio` / `jbig2-imageio` add-ons on the classpath). Verify against a representative corpus rather than a few sample files.

**Thread-safety.** A `PDDocument` is not thread-safe; do not share one instance across threads. Parallelize across documents, one `PDDocument` per thread, not across pages of a single document.

**Text order and layout.** `setSortByPosition(true)` fixes the common "right characters, wrong order" complaint but is O(n log n) per page and can itself reorder multi-column layouts oddly. For tabular data, `PDFTextStripperByArea` (region-based extraction) is usually more reliable than global sorting. There is no built-in table model.

**Upgrade pain.** 1.8 → 2.0 (2016) and 2.0 → 3.0 (2023) were both source-incompatible. The 2→3 jump is the one most projects still face: `PDDocument.load` → `Loader.loadPDF`, `MemoryUsageSetting` removal, and package/CLI changes. Budget real time; it is not a version-bump.

## When to Use / When Not

**Use when:**
- You need a permissive (Apache-2.0) PDF library for closed-source or commercial software.
- Your work is extraction, splitting/merging, form-filling, signing, stamping, or rasterizing existing PDFs.
- You want a battle-tested parser that tolerates malformed real-world files.
- You are already on the JVM and want to avoid shelling out to native tools.

**Avoid when:**
- You need rich document *generation* — flowing text, tables, tagged/accessible PDF, PDF/UA — with minimal manual layout; iText or a reporting engine is far less work.
- You need HTML/CSS → PDF; use an HTML-to-PDF renderer instead of hand-drawing content streams.
- You are not on the JVM — bindings exist but native ecosystems (see alternatives) are more idiomatic.
- Reliable text extraction from scanned or glyph-encoded PDFs is the core requirement; you need OCR (Tesseract), not PDFBox alone.

## Alternatives

- itext/itext-java — richer high-level generation, tagging, PDF/A & PDF/UA; use when features outweigh AGPL/commercial licensing.
- LibrePDF/OpenPDF — LGPL/MPL fork of the last open iText 4; use when you want an iText-style API without AGPL.
- mozilla/pdf.js — in-browser rendering and light extraction in JavaScript; use for client-side viewing, not JVM back-ends.
- pymupdf/PyMuPDF — fast MuPDF-backed extraction and rendering; use when you're in Python and want speed over permissive purity (AGPL/commercial).
- apache/fop — XSL-FO → PDF for templated document generation; use when your source is structured XML, not imperative drawing.

## History

| Version | Date | Notes |
|---------|------|-------|
| (incubation) | 2008 | Donated to the Apache Incubator by Ben Litchfield. |
| TLP | 2009 | Promoted to Apache top-level project. |
| 1.0.0 | 2010 | First Apache release; FontBox/JempBox merged in. |
| 1.8.x | 2013–2016 | Long-lived stable line on Java 5/6. |
| 2.0.0 | 2016-03 | Major API rewrite; new rendering and parsing engine[^4]. |
| 3.0.0 | 2023-08 | `Loader` API, module cleanup, JempBox removed, Java 8 baseline[^4]. |

## References

[^1]: Apache PDFBox — project overview and About page. https://pdfbox.apache.org/
[^2]: Apache PDFBox README — build requires Java 11+ and Maven 3. https://github.com/apache/pdfbox
[^3]: Apache PDFBox README — encryption via the Java Cryptography Architecture and Bouncy Castle; export-control (ECCN 5D002) notice. https://github.com/apache/pdfbox
[^4]: Apache PDFBox — release notes and migration guides (2.0 / 3.0). https://pdfbox.apache.org/2.0/migration.html
[^5]: Apache PDFBox README — "Known Limitations and Problems": glyph-encoding extraction requires OCR; text sorting is off by default. https://github.com/apache/pdfbox

## Tags

java, jvm, pdf, document-processing, text-extraction, pdf-rendering, apache, library, fontbox, digital-signatures
