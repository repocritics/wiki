# Hopding/pdf-lib

> Pure-JavaScript library to create and, unusually, *modify* existing PDF documents in any JS runtime.

[GitHub repo](https://github.com/Hopding/pdf-lib) ·
[Official website](https://pdf-lib.js.org) ·
[License: MIT](https://github.com/Hopding/pdf-lib/blob/master/LICENSE.md)

## Overview

pdf-lib is a TypeScript library (first published 2017) for programmatically building and editing PDF files. Its two distinguishing claims are stated plainly in the README: it can *modify* existing documents, not just generate new ones, and it runs in "any modern JavaScript runtime" — Node, browser, Deno, and React Native — because it ships zero native dependencies and implements PDF parsing, image decoding, and font handling in pure JS[^1].

That portability is the whole design premise. Most of the JS PDF ecosystem is split between generation-only tools (pdfkit, jsPDF) and native/binary-backed editors (HummusJS, Ghostscript bindings) that only run in Node. pdf-lib occupies the middle: an editable low-level PDF object model that works client-side in a browser tab. The cost of that choice is that it does the hard PDF work itself in JS rather than delegating to a mature C library — so its coverage of the PDF spec is deliberately partial (see Limitations).

The most important thing an operator should know before adopting it: the project is effectively dormant. The last commit to `master` was 2024-07-17, and the latest npm release (1.17.1) predates that by years[^2]. As of 2026 there are 317 open issues and a large backlog of unmerged pull requests. It remains widely used and works well within its supported feature set, but do not expect bug fixes, spec-gap closures, or maintainer response. Treat it as stable-but-frozen infrastructure.

## Getting Started

```bash
npm install pdf-lib
# custom (non-standard) font embedding also needs:
npm install @pdf-lib/fontkit
```

```js
import { PDFDocument, StandardFonts, rgb } from 'pdf-lib';

const pdfDoc = await PDFDocument.create();
const font = await pdfDoc.embedFont(StandardFonts.Helvetica);
const page = pdfDoc.addPage([600, 400]);

page.drawText('Hello from pdf-lib', {
  x: 50,
  y: 350,
  size: 24,
  font,
  color: rgb(0, 0.53, 0.71),
});

const pdfBytes = await pdfDoc.save(); // Uint8Array — write to disk or download
```

Nearly every entry-point (`create`, `load`, `embedFont`, `embedPng`, `save`) is `async` and returns a Promise, even when no I/O occurs — a consistency choice, not always a real await point.

## Architecture / How It Works

Underneath the friendly `PDFDocument` facade is a full low-level PDF object model. A document is a `PDFContext` holding indirect objects keyed by reference: `PDFDict`, `PDFArray`, `PDFName`, `PDFNumber`, `PDFString`/`PDFHexString`, `PDFStream`, and `PDFRef`. This is the actual PDF file structure exposed as typed JS objects, and advanced users can reach it via `pdfDoc.context` to hand-edit dictionaries the high-level API doesn't cover.

`PDFDocument.load(bytes)` runs a hand-written parser over the cross-reference table (or xref streams) and object graph; `save()` serializes the whole context back to bytes. Notably, `save()` performs a *full rewrite*, not a PDF incremental update — it renumbers and re-emits the object stream. This is why any pre-existing digital signature is invalidated on save, and why pdf-lib cannot round-trip a signed document.

Fonts come in two paths. The 14 Standard Fonts (Helvetica, Times, Courier, Symbol, ZapfDingbats) use built-in AFM metrics and require no embedding — but they are WinAnsi-encoded and cannot render most non-Latin text. Anything else (TrueType/OpenType, Unicode, CJK, emoji) goes through `@pdf-lib/fontkit`, which must be explicitly registered: `pdfDoc.registerFontkit(fontkit)`. fontkit also subsets embedded fonts to shrink output. Image embedding (`embedPng`, `embedJpg`) uses pure-JS decoders; other formats must be converted first. Forms are handled through an AcroForm layer (`getForm()`) exposing text fields, checkboxes, radio groups, dropdowns, option lists, and buttons, plus `flatten()` to bake field appearances into page content.

## Production Notes

**Maintenance status is the headline risk.** Last release 1.17.1 (npm) with no tagged releases since; `master` last touched 2024-07-17[^2]. 317 open issues. If you hit a spec gap, you own it — patch via a fork or the low-level `context` API. Several community forks exist specifically to add encryption/signing.

**Custom fonts fail silently if fontkit isn't registered.** `embedFont(customBytes)` throws or misbehaves without a prior `registerFontkit(fontkit)`. This is the single most common first-use error.

**Standard fonts are WinAnsi-only.** Drawing Korean, Japanese, Chinese, Arabic, or even smart quotes / emoji with a standard font produces `WinAnsi cannot encode` errors. The fix is always: embed a real font that covers the glyphs.

**No text extraction.** pdf-lib cannot read text content out of a page — there is no content-stream text parser. For extraction/search use pdf.js. This surprises people who assume "modify" implies "read."

**Encryption is essentially unsupported.** You cannot save an encrypted PDF. Loading an encrypted one requires `PDFDocument.load(bytes, { ignoreEncryption: true })`, which does not actually decrypt content and can yield corrupt output. Digital signing is not supported natively either (and `save()` breaks existing signatures).

**Everything is in memory.** The entire document graph is held in RAM and rewritten on save; there is no streaming/partial-write path. Multi-hundred-MB PDFs, or high-concurrency server workloads generating large files, can exhaust the heap. Size and rate-limit accordingly.

**Form field appearances need refreshing with custom fonts.** After setting field text with an embedded font, call `form.updateFieldAppearances(font)` (or flatten) so viewers render the intended glyphs; otherwise fields may show the default appearance.

## When to Use / When Not

**Use when:**
- You need to edit, stamp, fill forms in, merge, or split *existing* PDFs from JavaScript.
- The code must run client-side in a browser, or in Deno / React Native where native bindings aren't an option.
- You want a dependency-free, MIT-licensed library with a stable, well-documented API.

**Avoid when:**
- You need to extract or search text from PDFs — use pdf.js.
- You need encryption, decryption, or digital signatures out of the box.
- You need an actively maintained library with responsive upstream support and ongoing spec coverage.
- You're only ever *generating* new PDFs server-side and want streaming output — pdfkit is a better fit.

## Alternatives

- foliojs/pdfkit — generation-only with a streaming API; use instead when you never need to modify existing files and want lower memory use on the server.
- mozilla/pdf.js — rendering and text extraction; use instead when you need to *read/display* PDFs rather than author them.
- parallax/jsPDF — lightweight client-side generation; use instead for simple browser-side "print to PDF" output without an object model.
- diegomura/react-pdf — declarative React renderer for generating documents; use instead when you want JSX-driven layout rather than manual coordinates.
- julianhille/MuhammaraJS — native (HummusJS successor) Node editor; use instead when you need encryption, signing, or true incremental updates and can accept Node-only + native builds.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-09 | First published; create-and-modify PDFs in JS. |
| 1.0.0 | 2020 | Major rewrite of the public API; new low-level object model. See docs/MIGRATION.md[^3]. |
| 1.17.1 | ~2021 | Latest npm release; no tagged releases since. |
| — | 2024-07-17 | Last commit to `master`; project effectively dormant thereafter[^2]. |

## References

[^1]: pdf-lib README — "Create and modify PDF documents in any JavaScript environment. Tested in Node, Browser, Deno, and React Native." https://github.com/Hopding/pdf-lib
[^2]: GitHub repository metadata (fetched 2026-07): 8,539 stars, 900 forks, 317 open issues, MIT license, default branch `master`, last push 2024-07-17. https://github.com/Hopding/pdf-lib
[^3]: pdf-lib v1.0.0 migration guide. https://github.com/Hopding/pdf-lib/blob/master/docs/MIGRATION.md

## Tags

typescript, javascript, pdf, pdf-generation, pdf-manipulation, document, browser, nodejs, deno, forms, library, dormant
