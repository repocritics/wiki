# mholt/PapaParse

> A dependency-free, RFC 4180-oriented CSV parser for the browser and Node.js, built around graceful handling of large and malformed input.

[GitHub repo](https://github.com/mholt/PapaParse) ·
[Official website](https://www.papaparse.com) ·
[License: MIT](https://github.com/mholt/PapaParse/blob/master/LICENSE)

## Overview

Papa Parse is a single-file JavaScript library for reading and writing delimited text (CSV, TSV, and arbitrary delimiters). It was written by Matthew Holt — later known for the Caddy web server — and has been maintained since 2013[^1]. Its distinguishing goal is not raw throughput but tolerance: it parses in the browser against `File` objects, streams files larger than memory, runs off the main thread in a Web Worker, and returns errors as data rather than throwing, so a malformed row does not abort the whole parse.

The library has no dependencies (not even jQuery, despite historical optional integration) and ships as one UMD file that works via `<script>` tag, npm, or ES import. In practice it is the default CSV reader for browser front-ends — file-upload flows, in-browser data tools, spreadsheet importers — because it is one of the few parsers that correctly handles quoted fields containing embedded newlines and commas while still working on a `FileReader` stream.

The defining tension is browser-first design versus server workloads. On the browser it is close to unrivaled; on the server, Node-specific streaming CSV libraries (`csv-parse`, `fast-csv`) offer more back-pressure control and lower overhead. Several of Papa's headline options — `worker`, `download`, `withCredentials`, chunk-size tuning — are browser-only and silently unavailable under Node[^2].

## Getting Started

```shell
npm install papaparse
```

```js
import Papa from 'papaparse';

// Parse a string, with header row and type coercion
const result = Papa.parse('id,active\n1,true\n2,false', {
  header: true,
  dynamicTyping: true,
  skipEmptyLines: true,
});
console.log(result.data);   // [{ id: 1, active: true }, { id: 2, active: false }]
console.log(result.errors); // [] — errors are returned, not thrown

// Reverse: JSON back to CSV
const csv = Papa.unparse([{ id: 1, active: true }, { id: 2, active: false }]);
```

Streaming a large file in the browser without loading it all into memory:

```js
Papa.parse(fileInput.files[0], {
  worker: true,          // off the main thread
  step: (row) => { /* handle one row at a time */ },
  complete: () => { /* done */ },
});
```

Node streaming via `.pipe`:

```js
const Papa = require('papaparse');
const fs = require('fs');
fs.createReadStream('big.csv')
  .pipe(Papa.parse(Papa.NODE_STREAM_INPUT, { header: true }))
  .on('data', (row) => { /* ... */ })
  .on('end', () => { /* ... */ });
```

## Architecture / How It Works

Papa Parse is a hand-written character-scanning parser, not a regex or generated one. The core `Parser` walks the input tracking quote state, so it correctly treats delimiters and newlines that appear inside quoted fields as literal data — the RFC 4180 rule that trips up naive `split(',')` implementations.

Input handling is abstracted over several **streamer** types selected by what you pass to `Papa.parse`: a string is parsed directly; a browser `File`/`Blob` uses `FileReader` in chunks; a URL string with `download: true` fetches over HTTP in chunks; and `Papa.NODE_STREAM_INPUT` returns a Node transform stream. Each streamer feeds fixed-size chunks (`Papa.LocalChunkSize`, `Papa.RemoteChunkSize`) to the same core parser, which is how the library parses files larger than available memory.

Two result-delivery modes exist and they matter for memory:

- **Whole-file** — omit `step`/`chunk` and the full parsed array is buffered and handed to `complete` (or returned synchronously for strings). Simple, but the entire dataset lives in memory at once.
- **Streaming** — provide a `step` (per-row) or `chunk` (per-chunk) callback and rows are emitted incrementally and can be garbage-collected. This is the only safe path for files that do not fit in memory. `pause()`, `resume()`, and `abort()` are available on the parser handle inside these callbacks.

**Worker mode** (`worker: true`, browser only) spawns a Web Worker running the same code, so parsing a large file does not freeze the UI. The cost is that every row crosses the worker boundary via `postMessage` structured-clone serialization, which adds overhead and rules out passing non-cloneable callbacks into the parse.

`Papa.unparse` is the inverse path: it takes an array of objects or arrays and builds a CSV string, inferring columns from the first object's keys (or an explicit `fields` list). It is a string builder, not a stream — it constructs the entire output in memory.

## Production Notes

**`dynamicTyping` is a data-corruption footgun.** It coerces values that *look* numeric or boolean into JS numbers/booleans. That strips leading zeros from zip codes and phone numbers ("01234" → 1234), can exceed `Number.MAX_SAFE_INTEGER` on long IDs and lose precision, and converts the strings `"true"`/`"false"`. For identifier-like columns, leave it off or scope it per-column via a function.

**Fast mode trades correctness for speed.** Papa auto-enables (or you can force) a fast path that skips quote handling entirely. It is only safe when the data contains no quoted fields; on quoted input it will mis-split. Do not force `fastMode: true` on untrusted or mixed data.

**Errors are in `results.errors`, not exceptions.** A malformed file parses "successfully" and reports problems (`TooFewFields`, `TooManyFields`, `MissingQuotes`, etc.) as row-scoped entries. Code that only checks for a thrown error will silently accept bad data — always inspect `results.errors`.

**Whole-file parse will OOM on big inputs.** The synchronous/whole-file mode buffers everything. Anything beyond tens of MB should use `step`/`chunk` streaming, and in the browser `worker: true` on top of that. There is no automatic switch — choosing the wrong mode is the most common scaling mistake.

**`header: true` quirks.** Duplicate header names collide (later columns overwrite earlier ones in the row object). `transformHeader` can normalize names, but empty or whitespace headers still produce awkward keys. If column order or duplicates matter, parse as arrays and map headers yourself.

**`skipEmptyLines` has three behaviors.** `false` (keep), `true` (drop fully empty lines), and `'greedy'` (drop lines that are only whitespace/delimiters). Whitespace-only rows survive `true` but not `'greedy'` — a frequent source of stray empty rows downstream.

**Node feature gaps.** `worker`, `download`, `withCredentials`, and the chunk-size constants are no-ops under Node[^2]. For server-side streaming pipelines with real back-pressure, a Node-native parser is often the better fit; Papa's Node stream works but is a secondary path.

## When to Use / When Not

**Use when:**
- You are parsing CSV in the browser, especially from a `<input type="file">` upload.
- You need to handle files larger than memory via `step`/`chunk` streaming.
- Input may be malformed and you want per-row errors instead of a hard failure.
- You want one zero-dependency file that works in browser and Node without a build step.

**Avoid when:**
- You are building a heavy server-side ETL pipeline — `csv-parse` / `fast-csv` give finer streaming and back-pressure control.
- You need spreadsheet formats (XLSX, ODS) — Papa only does delimited text; use a spreadsheet library.
- You need maximum raw parse throughput on the server and control the input format — a specialized Node parser will usually win.
- Your data has identifier columns and you were about to enable `dynamicTyping` globally.

## Alternatives

- adaltas/node-csv (`csv-parse`) — Node-first, streaming-native with strong back-pressure; use instead when the workload is a server pipeline, not a browser upload.
- C2FO/fast-csv — combined Node CSV parse and format with a streaming API; use when you want both directions in one Node-oriented package.
- d3/d3-dsv — minimal, fast CSV/TSV parse and format; use when you only need in-memory parsing (often already present in a D3 app) and not streaming or workers.
- SheetJS/sheetjs — use when you actually need spreadsheets (XLSX/ODS/XLS) rather than plain delimited text.
- Keyang/node-csvtojson — use for Node CSV-to-JSON conversion with a streaming, promise-friendly API.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2013 | First release as "jQuery Parse", browser-focused[^1]. |
| 4.0 | 2016 | Renamed Papa Parse; Node.js support added, jQuery dependency dropped[^3]. |
| 5.0 | ~2018–2019 | Dropped legacy browser support; `NODE_STREAM_INPUT` and streaming refinements[^3]. |
| 5.x | ongoing | Active maintenance; ~13.5k stars, last pushed 2026-07 — steady bug-fix cadence rather than rapid feature churn. |

## References

[^1]: Papa Parse homepage and documentation. https://www.papaparse.com/docs
[^2]: README, "Papa Parse for Node" — lists `LocalChunkSize`, `RemoteChunkSize`, `download`, `withCredentials`, and `worker` as unavailable in Node. https://github.com/mholt/PapaParse#papa-parse-for-node
[^3]: Release history. https://github.com/mholt/PapaParse/releases

## Tags

javascript, csv, csv-parser, delimited-text, browser, nodejs, streaming, data-import, rfc-4180, zero-dependency
