# exceljs/exceljs

> Read, manipulate and write XLSX and CSV spreadsheet data and styles in pure JavaScript — reverse-engineered from the Office Open XML format.

[GitHub repo](https://github.com/exceljs/exceljs) ·
[License: MIT](https://github.com/exceljs/exceljs/blob/master/LICENSE)

## Overview

ExcelJS is a Node.js (and browser) library for creating and reading `.xlsx`
workbooks — cells, styles, formulas, merged ranges, images, data validation,
conditional formatting, and pivot tables (with limitations) — plus CSV I/O. It
was written by Guyon Roche and is, per the README, "reverse engineered from
Excel spreadsheet files as a project," which frames its character: it targets
the Office Open XML (OOXML) SpreadsheetML surface directly rather than wrapping
a native Excel engine.[^1]

The library occupies a specific niche: unlike CSV-only writers, it produces
real styled `.xlsx` files with fidelity Excel will open cleanly; unlike the
broadest-coverage parsers, it deliberately supports only the modern
XML-based `.xlsx` format (no legacy binary `.xls`, no `.ods`). Its defining
tension is between a rich, mutable in-memory object model (convenient, but
memory-heavy on large sheets) and a separate streaming API bolted alongside it
for volume work. The two models do not share the same feature set, which is the
single most common source of surprise for new users.

A second tension is maintenance velocity. ExcelJS is widely depended upon (over
15k stars, ~2k forks, ~180 watchers) but its release cadence has slowed
markedly: the last tagged npm release was 4.4.0, and while the `master` branch
saw commits into 2025, roughly 795 issues remain open. Treat it as a mature,
feature-complete-enough library in low-activity maintenance rather than an
actively evolving one.[^2]

## Getting Started

```bash
npm install exceljs
```

```javascript
const ExcelJS = require('exceljs');

async function run() {
  const workbook = new ExcelJS.Workbook();
  const sheet = workbook.addWorksheet('Report');

  sheet.columns = [
    { header: 'Name', key: 'name', width: 24 },
    { header: 'Total', key: 'total', width: 12, style: { numFmt: '#,##0.00' } },
  ];

  sheet.addRow({ name: 'Alice', total: 1234.5 });
  const row = sheet.addRow({ name: 'Bob', total: 987 });
  row.getCell('name').font = { bold: true };

  await workbook.xlsx.writeFile('report.xlsx');
}

run();
```

Reading is symmetric: `await workbook.xlsx.readFile('report.xlsx')` then walk
`workbook.eachSheet(...)` and `sheet.eachRow(...)`. In the browser, read from a
buffer/blob with `workbook.xlsx.load(arrayBuffer)`.

## Architecture / How It Works

An `.xlsx` file is a ZIP archive of XML parts (`workbook.xml`, one
`sheetN.xml` per sheet, a shared-strings table, `styles.xml`, drawings,
etc.). ExcelJS builds on this directly: **jszip** handles the ZIP container and
a SAX-style XML parser (**saxes**) streams the parts, while a family of
per-part "xform" classes marshal XML to and from the JS object model. CSV I/O
is delegated to **fast-csv**, and dates are handled via **dayjs**.[^3]

The public surface is a document object model. A `Workbook` holds
`Worksheet`s; a worksheet exposes `getRow`, `getCell`, `columns`, `mergeCells`,
styling buckets, and iteration helpers. Cell values are tagged unions — a cell
can be a number, string, date, hyperlink, formula, rich text, boolean, or
error, each with its own value shape. Styles (font, fill, border, alignment,
numFmt) are properties on cells, rows, and columns, with column/row styles
acting as defaults that individual cells override.

Critically, **ExcelJS does not evaluate formulas.** When you set a cell to
`{ formula: 'A1+A2' }`, the library writes the formula string into the sheet
and stores an optional cached `result`; it never computes the value itself.
Excel recalculates on open (optionally forced via
`workbook.calcProperties.fullCalcOnLoad = true`), but any consumer reading the
file programmatically sees only whatever cached `result` you supplied. This is
by design — it is a serializer, not a spreadsheet engine — but it routinely
trips up people expecting computed output.

For volume, a parallel **streaming** layer exists under
`ExcelJS.stream.xlsx.WorkbookWriter` / `WorkbookReader`. The writer commits
rows to disk incrementally (via `row.commit()` / `sheet.commit()`) so the whole
workbook never lives in memory; the reader emits rows as it parses. These
streaming classes are separate implementations and support a *subset* of the
in-memory API — some styling, image, and post-hoc mutation operations are
unavailable once a row is committed.

## Production Notes

**Memory is the dominant operational concern.** The default `Workbook` holds
the entire sheet as live JS objects; generating or reading workbooks with
hundreds of thousands of rows can exhaust heap or run very slowly. The
mitigation is the streaming writer/reader — but adopting it means giving up the
random-access object model and some features. Budget for a rewrite, not a flag
flip, if you start on the in-memory API and hit a ceiling.

**Formulas are inert (see above).** If a downstream system reads your generated
file without opening it in Excel, populate `result` explicitly. Reading files
authored by Excel is fine — the cached results are already present — but
regenerating those files can stale the cache.

**`getWorksheet(id)` is a footgun.** Worksheet ids are not positional and do
not necessarily start at 1 (deleting a sheet leaves gaps); the README itself
warns against relying on numeric ids. Prefer `getWorksheet(name)`,
`eachSheet`, or the `worksheets` array.

**Style objects are shared by reference until serialized.** Assigning the same
style object to many cells and then mutating it can produce surprising
cross-cell effects; construct fresh style objects or rely on column/row-level
defaults.

**Bundle size and browser polyfills.** The browser build is large, and the ES5
`dist/es5` build has *implicit* peer dependencies on `core-js` and
`regenerator-runtime` polyfills that ExcelJS no longer bundles — you must add
them yourself for older runtimes. IE11 additionally needs a unicode-regex
polyfill.

**Maintenance risk.** With no recent tagged release and a large open-issue
backlog, do not expect prompt fixes for edge-case OOXML compatibility bugs.
Pin your version, keep a reproduction handy, and be prepared to patch or
vendor if you hit a parser edge case with an unusual Excel-authored file.

## When to Use / When Not

**Use when:**
- You need to generate styled `.xlsx` files (fonts, fills, borders, number
  formats, merged cells, images) that open cleanly in Excel/LibreOffice.
- You are reading or writing modern `.xlsx` and want a mature, dependency-light,
  MIT-licensed option with TypeScript definitions included.
- You have large datasets and can commit to the streaming API from the start.

**Avoid when:**
- You need formula *evaluation* — ExcelJS never computes results; pair it with a
  calc engine or precompute values.
- You must support legacy `.xls` (BIFF), `.ods`, or the widest matrix of formats.
- You only ever touch CSV — a dedicated CSV library is lighter.
- You require an actively maintained project with a responsive issue tracker.

## Alternatives

- SheetJS/sheetjs (the `xlsx` package) — use instead when you need the widest
  format coverage (`.xls`, `.ods`, `.xlsb`, HTML) or robust reading of
  messy real-world files; styling/writing fidelity largely lives in its paid Pro tier.
- dtjohnson/xlsx-populate — use instead when you want to edit an existing
  template and preserve its formatting, rather than build workbooks from scratch.
- exceljs is CSV-capable, but for CSV-only pipelines use C2FO/fast-csv (which
  ExcelJS itself uses) for a lighter, streaming-native dependency.
- dream-num/univer or the older luckysheet — use instead when you need an
  in-browser spreadsheet *UI/engine* with live recalculation, not just file I/O.
- writeexcel/write-excel-file — use instead for a smaller, schema-driven writer
  when you only need to emit simple styled `.xlsx` and value bundle size.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2015 | Early releases; core XLSX read/write and object model. |
| 1.x–3.x | 2016–2019 | Styling, streaming writer/reader, CSV, images, data validation added incrementally. |
| 4.0.0 | 2020 | Major line; dependency and build modernization. |
| 4.3.0 | 2021–2022 | Broadened OOXML feature coverage, type-definition fixes. |
| 4.4.0 | 2023 | Latest tagged npm release; last widely-shipped version.[^2] |

Per-release dates are omitted where not independently verified; consult the
repository's tags and CHANGELOG for authoritative timing.

## References

[^1]: ExcelJS README — "Read, manipulate and write spreadsheet data and styles to XLSX and JSON. Reverse engineered from Excel spreadsheet files as a project." https://github.com/exceljs/exceljs
[^2]: Repository metadata via GitHub API (fetched 2026-07-15): ~15,401 stars, ~2,013 forks, ~180 watchers, ~795 open issues, MIT license, default branch `master`, created 2014-12-09, last push 2025-01-21. https://github.com/exceljs/exceljs
[^3]: ExcelJS runtime dependencies (jszip for the ZIP container, saxes for XML parsing, fast-csv for CSV, dayjs for dates) per its `package.json`. https://github.com/exceljs/exceljs/blob/master/package.json

## Tags

javascript, nodejs, xlsx, spreadsheet, excel, ooxml, csv, streaming, file-io, browser
