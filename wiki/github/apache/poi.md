# apache/poi

> The Java library for reading and writing Microsoft Office file formats — the JVM's default answer for programmatic Excel, Word, and PowerPoint.

[GitHub repo](https://github.com/apache/poi) ·
[Official website](https://poi.apache.org/) ·
[License: Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0)

## Overview

Apache POI ("Poor Obfuscation Implementation" — the name is a deliberate joke about
reverse-engineering Microsoft's undocumented binary formats) is a pure-Java library for
reading and writing Office documents[^1]. It began in the early 2000s as a Jakarta project
to decode the OLE2 compound-document format behind `.xls`, and graduated to a top-level
Apache Software Foundation project in 2007. Today it covers both the legacy binary formats
(OLE2: `.xls`, `.doc`, `.ppt`) and the ZIP-of-XML OOXML formats (`.xlsx`, `.docx`, `.pptx`)
introduced with Office 2007.

The library is organized by a two-letter component naming scheme that leaks its history.
Components named `H**F` (HSSF, HWPF, HSLF) read and write the old **binary** formats;
components named `X**F` (XSSF, XWPF, XSLF) handle **OOXML**. A "common" layer (SS for
spreadsheets, WP for word processing, SL for slides) sits on top so callers can target one
API against both format families. Maturity is uneven and honest about it: Excel support
(SS = HSSF + XSSF + SXSSF) is production-grade; Word and PowerPoint are usable but
thinner; Visio, Publisher, and Outlook (HDGF/HPBF/HSMF) are best-effort.

The defining tension is memory. The default OOXML path parses an entire workbook into a
DOM-like object tree via Apache XMLBeans, which is convenient but scales with file size —
large spreadsheets routinely exhaust the heap. Almost every non-trivial POI production
story is really a story about which of POI's several APIs you picked to avoid loading the
whole document at once.

## Getting Started

```groovy
// Gradle — pull the OOXML module; it transitively brings core poi + xmlbeans
dependencies {
    implementation 'org.apache.poi:poi-ooxml:5.4.1'
    // add poi-scratchpad only if you need legacy .doc/.ppt (HWPF/HSLF)
}
```

```java
// Read a cell, then write a new .xlsx
import org.apache.poi.ss.usermodel.*;
import org.apache.poi.xssf.usermodel.XSSFWorkbook;
import java.io.*;

try (Workbook wb = WorkbookFactory.create(new File("in.xlsx"))) {
    Sheet sheet = wb.getSheetAt(0);
    String v = sheet.getRow(0).getCell(0).getStringCellValue();
    System.out.println(v);
}

try (Workbook out = new XSSFWorkbook();
     OutputStream os = new FileOutputStream("out.xlsx")) {
    Row row = out.createSheet("data").createRow(0);
    row.createCell(0).setCellValue("hello");
    out.write(os);
}
```

`WorkbookFactory.create` sniffs the format and returns HSSF (`.xls`) or XSSF (`.xlsx`)
transparently. That convenience is also the memory trap — see below.

## Architecture / How It Works

POI is really several stacked APIs, not one:

- **POIFS** — the OLE2 compound-file reader. Every binary Office file is a tiny in-file
  filesystem of named streams; POIFS is the layer that navigates it. HSSF/HWPF/HSLF sit on
  top.
- **OpenXML4J / OPC** — the Open Packaging Conventions layer: an OOXML file is a ZIP of XML
  parts with a relationship graph. XSSF/XWPF/XSLF are built on it, and on **Apache XMLBeans**,
  which generates Java classes from Microsoft's published XSD schemas.
- **HPSF** — document-property (metadata) reader shared across formats.

The XMLBeans dependency is the crux of POI's footprint. The generated schema classes are
enormous, which is why the OOXML support ships in two flavors: `poi-ooxml-lite` (the common
subset, the default transitive dependency) and `poi-ooxml-full` (every generated class, for
less common features). Picking the wrong one produces `ClassNotFoundException`s for schema
types at runtime rather than compile time.

For Excel specifically there are three write/read strategies with very different memory
profiles:

1. **XSSF** — full DOM in memory. Random access to any cell, but heap usage tracks file size.
2. **SXSSF** — a streaming *writer* that keeps only a sliding window of recent rows in memory
   and flushes older rows to temporary files on disk. It cannot read, and once a row scrolls
   out of the window you cannot go back and edit it.
3. **XSSF + SAX event API** (`XSSFReader` + a `SheetContentsHandler`) — a streaming *reader*
   that never builds the object tree. Much lower level: you handle SAX callbacks and resolve
   the shared-strings table yourself, but you can read arbitrarily large sheets.

Most heap-exhaustion incidents are caused by reaching for the friendly `WorkbookFactory` /
`Sheet` / `Row` / `Cell` API on a file that needed one of the streaming paths.

## Production Notes

- **OutOfMemoryError on large `.xlsx` is the number-one operator issue.** A 50 MB xlsx can
  expand to well over a gigabyte of heap under XSSF because XML and the shared-strings table
  inflate dramatically in object form. Budget the streaming reader for anything user-uploaded
  and unbounded in size.
- **SXSSF writes to a temp directory.** It needs writable scratch space (and disk headroom
  equal to the compressed output) and you must call `dispose()` to delete the temp files —
  forgetting it leaks disk. Its default row window is 100; widen it only if you truly need
  backward access, since that raises the memory floor.
- **Zip-bomb and decompression guards will reject legitimate large files.** POI enforces a
  minimum inflate ratio (`ZipSecureFile.setMinInflateRatio`) and a maximum byte-array size
  (`IOUtils.setByteArrayMaxOverride`); highly repetitive real spreadsheets can trip the
  "Zip bomb detected" exception and require deliberately raising those thresholds[^2].
- **Logging is Log4j 2 API since POI 5.0.0.** POI dropped its old `POILogger` and now logs
  through `log4j-api`[^3]. It does *not* pull `log4j-core`, so POI was never exposed to
  Log4Shell (CVE-2021-44228) — but the dependency name caused significant false-alarm churn
  during that period, and you must supply your own logging backend or logs vanish silently.
- **Java baseline moves.** POI 4.x/5.x require Java 8; the `trunk` branch targeting 6.0.0
  requires Java 11[^1]. `module-info` descriptors exist but the multi-jar, XMLBeans-heavy
  dependency graph makes clean JPMS/`jlink` setups fiddly.
- **Formula evaluation is opt-in and incomplete.** Cached formula results are read for free,
  but recomputation requires `FormulaEvaluator`, and POI implements a large-but-not-complete
  subset of Excel's functions — unsupported functions throw at evaluation time.
- **`.doc`/`.ppt` (HWPF/HSLF) are second-class.** They live in `poi-scratchpad`, round-trip
  imperfectly, and are best treated as read-oriented. New work targeting `.docx`/`.pptx`
  (XWPF/XSLF) is on firmer ground.

## When to Use / When Not

**Use when:**
- You need to read or write real Excel/Word/PowerPoint files on the JVM with format fidelity,
  including styles, formulas, charts, and both legacy and OOXML variants.
- You need random access to arbitrary cells, or to author richly formatted documents.
- You want a permissively licensed, ASF-governed dependency with no per-seat cost.

**Avoid when:**
- Your only job is high-throughput, low-memory streaming of simple tabular `.xlsx` — a
  purpose-built library will use a fraction of the heap with far less API surface.
- You need server-side rendering to PDF/image with pixel-perfect Office fidelity; POI does not
  render, and a commercial engine is the realistic path.
- You only handle CSV or a single simple format — POI's dependency weight (XMLBeans,
  commons-compress, commons-io, log4j-api) is overkill.

## Alternatives

- dhatim/fastexcel — use instead when you only need fast, low-memory reading and writing of
  straightforward `.xlsx` and can forgo POI's full styling/formula surface.
- alibaba/easyexcel — use when you want annotation-driven, low-memory Excel mapping on the
  JVM; it wraps POI's SAX layer to cut heap use on large files.
- pjfanning/excel-streaming-reader — use when you want a `Workbook`-like streaming *reader*
  over huge `.xlsx` without hand-writing SAX handlers (it wraps POI).
- jxls-project/jxls — use when the task is templated report generation (Excel templates +
  data binding) rather than low-level cell manipulation.
- Aspose.Cells (commercial, not on GitHub) — use when you need rendering, conversion fidelity,
  and vendor support and can pay per-developer licensing.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2007 | Graduated to a top-level Apache Software Foundation project. |
| 4.0.0 | 2018-09 | Java 8 minimum; removed long-deprecated APIs. |
| 4.1.0 | 2019-04 | Continued OOXML and streaming improvements. |
| 5.0.0 | 2021-01 | Switched logging to the Log4j 2 API; XMLBeans bundled; `module-info`[^3]. |
| 5.2.0 | 2022-01 | Dependency and format-coverage updates on the Java 8 baseline. |
| 5.3.0 | 2024-04 | Ongoing fixes; Java 8 still supported. |
| 5.4.x | 2025 | Current 5.x line. |
| 6.0.0 | trunk (dev) | Java 11 minimum; in development on the `trunk` branch[^1]. |

## References

[^1]: Apache POI README and project site — components, module list, and Java baselines. https://poi.apache.org/
[^2]: Apache POI FAQ, "Zip bomb detected" / decompression limits (`ZipSecureFile`, `IOUtils.setByteArrayMaxOverride`). https://poi.apache.org/help/faq.html
[^3]: Apache POI 5.0.0 release notes — migration to the Log4j 2 API logging facade. https://poi.apache.org/changes.html

## Tags

java, excel, xlsx, office, ooxml, spreadsheet, docx, pptx, file-format, jvm, apache, document-processing
