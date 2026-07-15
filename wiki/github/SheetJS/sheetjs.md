# SheetJS/sheetjs

> Pure-JavaScript spreadsheet toolkit that reads and writes almost any spreadsheet format through a single in-memory object model.

[GitHub repo](https://github.com/SheetJS/sheetjs) ·
[Official website](https://sheetjs.com/) ·
[License: Apache-2.0](https://github.com/SheetJS/sheetjs/blob/github/LICENSE)

## Overview

SheetJS (published on npm as `xlsx`, historically "js-xlsx") is a dependency-free JavaScript library that parses and generates spreadsheets. Its organizing idea is a single normalized workbook model — the "Common Spreadsheet Format" (CSF) — that every supported file type reads into and writes out of. XLSX/XLSM, XLSB, the legacy binary XLS (BIFF2–8), SpreadsheetML 2003, ODS, CSV, DBF, and HTML tables all decode to the same cell objects, so application code deals with one shape regardless of source[^1]. It runs in browsers, Node, Deno, and Bun with no native bindings.

The defining tension is the split between the free Community Edition and the commercial SheetJS Pro. CE (Apache-2.0) is data-in / data-out: it faithfully preserves cell values, types, formulas-as-text, and number formats, and it is genuinely battle-tested against malformed real-world files. It does **not** evaluate formulas, and its handling of styling, images, charts, and PivotTables is intentionally minimal — those are Pro features[^2]. Teams routinely adopt CE for import/export and only later discover that "make the output look like the template" is out of scope.

The second thing to know before adopting: as of 2026 the canonical source and package distribution have **moved off GitHub and npm**. The GitHub repository is a frozen mirror (its default `github` branch is a redirect stub, which is why GitHub reports no primary language and no recent pushes); active development lives at git.sheetjs.com, and current releases are published only through the SheetJS CDN[^3]. The GitHub stars here are historical accumulation, not a signal of recent GitHub activity.

## Getting Started

The version on the public npm registry is frozen at `0.18.5`. Current releases install from the SheetJS CDN tarball[^3]:

```bash
# current release (recommended) — from SheetJS CDN
npm i --save https://cdn.sheetjs.com/xlsx-0.20.3/package/xlsx.tgz
# frozen legacy on the public npm registry (0.18.5, do not use for new work)
# npm i xlsx
```

```js
import * as XLSX from "xlsx";
import fs from "fs";

// read: any supported format -> normalized workbook
const buf = fs.readFileSync("input.xlsx");
const wb = XLSX.read(buf);                       // detects format
const ws = wb.Sheets[wb.SheetNames[0]];

// worksheet -> array of row objects (header row = keys)
const rows = XLSX.utils.sheet_to_json(ws);

// build a new workbook from JSON and write xlsx
const out = XLSX.utils.book_new();
XLSX.utils.book_append_sheet(out, XLSX.utils.json_to_sheet(rows), "Sheet1");
XLSX.writeFile(out, "output.xlsx");
```

## Architecture / How It Works

Everything revolves around CSF, a plain-object representation with no classes to serialize:

- A **workbook** is `{ SheetNames: string[], Sheets: { [name]: Worksheet } }`.
- A **worksheet** is an object keyed by A1-style addresses (`ws["B2"]`) plus metadata keys: `!ref` (used range), `!cols`, `!rows`, `!merges`.
- A **cell** is `{ t, v, w, f, z }` — `t` is the type (`n` number, `s` string, `b` boolean, `d` date, `e` error, `z` stub), `v` the raw value, `w` the formatted text, `f` the formula string, `z` the number format.

Format support is a matrix of parser/writer pairs that all target CSF, so adding a format does not change consumer code. Number formatting is handled by a bundled implementation of the spreadsheet format-code grammar (the `SSF` module) rather than a date/number library.

The `XLSX.utils` namespace is where most applications actually live: `sheet_to_json` / `json_to_sheet`, `aoa_to_sheet` (array-of-arrays), `sheet_to_csv`, `sheet_to_html`, `book_new`, `book_append_sheet`, `encode_cell` / `decode_range`. These are deliberately dumb transforms over CSF, which is why the library composes well but also why higher-level concerns (schemas, validation, typed columns) are left to the caller.

Because CE preserves rather than computes, a formula cell round-trips its `f` text and last-cached `v`, but SheetJS CE will not recalculate it. Reading a file where the producing application never cached results can yield cells with a formula and no value.

## Production Notes

**Distribution is the biggest operational surprise.** `npm i xlsx` silently installs the frozen `0.18.5`. Any current features, and importantly current security fixes, only exist on the CDN tarballs[^3]. Lockfiles, private registry mirrors, and "just run npm install" CI all need to be pointed at the CDN URL explicitly, and Dependabot/Renovate will not see the newer versions.

**Security advisories interact badly with the npm freeze.** CE has had a prototype-pollution advisory (CVE-2023-30533) and a ReDoS advisory (CVE-2024-22363); the fixes ship in CDN releases (0.19.3 and 0.20.2 respectively), not on the public npm registry[^4]. The practical consequence: `npm audit` flags `xlsx` and reports "no fix available," because the fixed versions are not on the registry it queries. Resolution is to install the CDN tarball, not to wait for an npm patch.

**Large-file memory.** `XLSX.read` builds the entire workbook in memory as JS objects; multi-hundred-MB XLSX or wide sheets can blow heap. `dense: true` switches worksheets to array-of-arrays storage that is more memory-efficient for large grids. There is no streaming reader in CE — for truly large XLSX, a streaming-specific library or Pro is the answer.

**Untrusted input is a real threat surface.** Spreadsheet formats are complex binary/XML containers; parsing attacker-supplied files has historically been where the CVEs came from. Parse untrusted uploads in an isolated process, cap sizes, and stay current with CDN releases.

**Styling/images/formula-eval are not bugs to file — they are Pro.** Recurring frustration in the tracker is CE dropping cell colors, fonts, and embedded images on write. That is by design; needing it means either SheetJS Pro or a formatting-oriented alternative[^2].

**Dates are configuration-dependent.** By default numeric cells that are dates come back as numbers; `cellDates: true` returns JS `Date` objects. The 1900/1904 epoch and timezone handling are common sources of off-by-one-day bugs.

## When to Use / When Not

**Use when:**
- You need to import or export tabular data across many spreadsheet formats with one code path.
- You want zero native dependencies and the same API in browser, Node, Deno, and Bun.
- The job is data movement (JSON ⇆ sheet), not visual fidelity.
- You must ingest messy, legacy, real-world files (old XLS, oddball CSV) that other libraries reject.

**Avoid when:**
- You need to produce styled workbooks — fonts, fills, images, charts, PivotTables (use a formatting-first library or SheetJS Pro).
- You need server-side formula evaluation.
- You only ever touch CSV; a dedicated CSV parser is smaller and faster.
- Your supply-chain policy requires everything from the public npm registry with working `npm audit` — the CDN distribution model will fight you.

## Alternatives

- exceljs/exceljs — use when you need to write styled XLSX (fonts, fills, borders, images) and are content with xlsx-only, accepting heavier memory use.
- mholt/PapaParse — use when the entire job is CSV; it is smaller, faster, and streams.
- dtjohnson/xlsx-populate — use when you must edit an existing styled template in place without discarding its formatting.
- catamphetamine/read-excel-file — use when you want schema-validated, typed row parsing rather than a generic cell model.
- SheetJS Pro (commercial) — use when you need the same CSF core plus styling, images, charts, and formula evaluation.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2012 | Public repository created; pure-JS XLS/XLSX parsing[^1]. |
| 0.18.5 | 2022 | Final release published to the public npm registry[^3]. |
| 0.19.x | 2023 | Distribution moves to the SheetJS CDN; prototype-pollution fix (0.19.3)[^4]. |
| 0.20.x | 2024 | Continued CDN-only releases; ReDoS fix (0.20.2)[^4]. |
| mirror freeze | 2024-04 | GitHub mirror frozen; canonical source at git.sheetjs.com[^3]. |

## References

[^1]: SheetJS documentation, "Common Spreadsheet Format" and supported file formats. https://docs.sheetjs.com/docs/csf/
[^2]: SheetJS, Community Edition vs. Pro feature comparison. https://sheetjs.com/pro
[^3]: SheetJS repository README and installation docs — canonical source at git.sheetjs.com, releases via the SheetJS CDN. https://docs.sheetjs.com/docs/getting-started/installation/
[^4]: GitHub Advisory Database — SheetJS `xlsx` advisories (CVE-2023-30533 prototype pollution; CVE-2024-22363 ReDoS). https://github.com/advisories?query=xlsx

## Tags

javascript, spreadsheet, xlsx, excel, csv, data-import, data-export, parser, nodejs, browser, file-format
