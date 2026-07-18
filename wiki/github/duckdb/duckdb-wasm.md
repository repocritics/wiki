# duckdb/duckdb-wasm

> DuckDB compiled to WebAssembly — a full in-process OLAP SQL engine that runs
> inside the browser tab, querying Parquet/CSV/JSON over HTTP without a server.

[GitHub repo](https://github.com/duckdb/duckdb-wasm) ·
[Official shell](https://shell.duckdb.org) ·
[License: MIT](https://github.com/duckdb/duckdb-wasm/blob/main/LICENSE)

## Overview

DuckDB-Wasm is the official WebAssembly build of DuckDB, the in-process
analytical SQL database. It was developed at TU Munich / CWI and announced in
October 2021[^1], with a VLDB 2022 paper describing the design[^2]. The pitch
is unusual and real: a columnar, vectorized OLAP engine with full SQL —
window functions, joins, Parquet and CSV readers — executing entirely
client-side. It reads and writes Apache Arrow natively, which is how query
results cross the Wasm/JS boundary. As of mid-2026 the build tracks DuckDB
v1.5.4[^3].

The audience is data-tool builders: notebooks (Observable, Evidence, Mosaic),
BI dashboards that push computation to the client, and anything that wants
"query a 2 GB Parquet file on S3 from a static page" — the engine fetches
only the byte ranges it needs via HTTP range requests[^2].

The defining tension: a database engine designed around threads, mmap, and
unrestricted file I/O is running in the most restrictive mainstream sandbox
there is. Every rough edge — single-threaded default execution, CORS
failures, the 32-bit memory ceiling, a reimplemented-in-JS HTTP stack —
traces back to that mismatch. At ~2.1k stars it is a small satellite of
duckdb/duckdb, but it is maintained by the DuckDB team itself and releases
track core closely.

## Getting Started

```bash
npm install @duckdb/duckdb-wasm apache-arrow
```

```ts
import * as duckdb from "@duckdb/duckdb-wasm";

// Pick the best bundle for this browser (mvp / eh / threaded)
const bundle = await duckdb.selectBundle(duckdb.getJsDelivrBundles());
const worker_url = URL.createObjectURL(
  new Blob([`importScripts("${bundle.mainWorker}");`], { type: "text/javascript" }),
);
const db = new duckdb.AsyncDuckDB(new duckdb.ConsoleLogger(), new Worker(worker_url));
await db.instantiate(bundle.mainModule, bundle.pthreadWorker);
URL.revokeObjectURL(worker_url);

const conn = await db.connect();
const result = await conn.query(`
  SELECT count(*) AS n
  FROM read_parquet('https://blobs.duckdb.org/data/stations.parquet')
`);
console.log(result.toArray());   // Arrow table -> JS objects
await conn.close();
```

## Architecture / How It Works

The C++ DuckDB core is compiled with Emscripten. Because browser capabilities
vary, the npm package ships several Wasm bundles and feature-detects at
runtime via `selectBundle`: a baseline MVP build, a build using native Wasm
exception handling, and a threaded build using SharedArrayBuffer[^4].

The engine runs inside a **Web Worker**; the main thread talks to it through
`AsyncDuckDB`, a message-protocol proxy. Query results are serialized as
Arrow IPC buffers and materialized as Arrow tables on the JS side — there is
no row-by-row cursor protocol, which keeps the boundary cheap for columnar
results and expensive for huge ones.

File access goes through a virtualized filesystem layer. Backends include
in-memory buffers (`registerFileBuffer`), browser `File` handles, HTTP(S)
URLs, and OPFS (origin-private file system) for persistent local storage.
The HTTP backend is the interesting part: the paper describes reading remote
Parquet via range requests, so `SELECT ... FROM 'https://...file.parquet'
LIMIT 10` fetches footer + needed row groups only[^2]. `LOAD httpfs` swaps in
a JavaScript reimplementation of DuckDB's native httpfs — same SQL surface,
different network behavior (requests are force-upgraded to HTTPS and subject
to CORS)[^3].

Extensions are dynamically loaded Wasm modules fetched from
extensions.duckdb.org or community-extensions.duckdb.org. Unlike native
DuckDB, which bundles core extensions (JSON, Parquet, ICU) into the binary,
DuckDB-Wasm autoloads them at runtime to keep initial download small —
`INSTALL x` is lazy and the actual fetch happens on `LOAD`[^3]. The repo
itself is polyglot: C++ engine glue, the TypeScript API package, and a
Rust-built terminal shell (shell.duckdb.org).

## Production Notes

- **CORS is the number-one support issue.** Every remote file the engine
  touches must be served with permissive CORS headers; S3 buckets and static
  hosts need explicit configuration. Errors surface as opaque I/O failures,
  not "CORS blocked".
- **Single-threaded by default.** The threaded build needs SharedArrayBuffer,
  which requires cross-origin isolation (COOP/COEP headers) on the embedding
  page[^5] and breaks third-party iframe embeds, so most deployments run one
  thread; the README still labels multithreading experimental[^3]. Expect a
  fraction of native DuckDB throughput.
- **Memory ceiling.** wasm32 caps the address space at 4 GB, with no
  spill-to-disk comparable to native out-of-core execution[^3]. Large
  joins/sorts fail with OOM rather than degrading.
- **Results are fully materialized.** `conn.query()` buffers the whole Arrow
  result. Use `conn.send()` for streaming batches, and put `LIMIT` on
  exploratory queries a user can type.
- **Bundle weight.** The Wasm module is several MB compressed before any
  extension; extensions add more (the README's extension demo transfers
  ~3.2 MB of extra Wasm)[^3]. Serve bundles from your own origin with
  long-lived caching rather than relying on jsDelivr in production.
- **Runtime network dependency.** Extension autoloading fetches from DuckDB's
  repositories at runtime; offline or locked-down environments must self-host
  the extension repository or preload extensions.
- **Bundler friction.** Worker + Wasm asset wiring differs per bundler;
  Vite/Webpack setups commonly need manual worker URL handling (per-bundler
  recipes in the docs)[^4].
- **Version skew.** npm versions do not equal DuckDB versions; check release
  notes for which core version a release embeds (currently 1.5.4)[^3].

## When to Use / When Not

**Use when:**
- You want interactive analytics on Parquet/CSV already on HTTP/S3, no backend.
- You build notebook/BI tools and want SQL + Arrow interchange in the client.
- Privacy or cost argues for client-side compute (data never leaves the browser).
- You need ad-hoc SQL over user-provided local files (drag-and-drop a CSV).

**Avoid when:**
- Working sets approach gigabytes or need out-of-core execution — use native
  DuckDB behind an API.
- You need OLTP-style writes, concurrency, or durable multi-user state.
- Your users are on low-end mobile; multi-MB Wasm + one thread is a poor fit.
- You cannot control CORS/COOP/COEP headers on your data sources and pages.

## Alternatives

- duckdb/duckdb — native engine; use when data exceeds browser memory or you
  control a backend.
- sql-js/sql.js — SQLite in Wasm; use for small relational/OLTP-style data
  where a lean engine beats OLAP features.
- electric-sql/pglite — Postgres in Wasm; use when you need Postgres
  semantics client-side rather than analytical throughput.
- finos/perspective — Wasm streaming pivot/viz engine; use for live-updating
  dashboards where the UI component matters more than general SQL.
- apache/arrow — Arrow JS alone; use when you can express transforms in code
  and only need columnar data handling, not SQL.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial release | 2021-10 | Public announcement on duckdb.org[^1]. Repo created 2021-05. |
| VLDB paper | 2022-09 | "DuckDB-Wasm: Fast Analytical Processing for the Web", PVLDB 15(12)[^2]. |
| 1.x npm line | 2022– | Package versioning decoupled from DuckDB core; each release embeds a specific core version. |
| Current | 2026-07 | Tracks DuckDB v1.5.4; OPFS persistence available; multithreading still experimental[^3]. |

## References

[^1]: André Kohn, "DuckDB-Wasm: Efficient Analytical SQL in the Browser" — 2021-10-29. https://duckdb.org/2021/10/29/duckdb-wasm.html
[^2]: Kohn, Moritz, Raasveldt, Mühleisen, Neumann, "DuckDB-Wasm: Fast Analytical Processing for the Web", PVLDB 15(12), 2022. https://www.vldb.org/pvldb/vol15/p3574-kohn.pdf
[^3]: duckdb-wasm README (core version basis, native-vs-Wasm differences). https://github.com/duckdb/duckdb-wasm#readme
[^4]: DuckDB docs, "DuckDB Wasm" client guide. https://duckdb.org/docs/stable/clients/wasm/overview
[^5]: MDN, SharedArrayBuffer security requirements (COOP/COEP). https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SharedArrayBuffer#security_requirements

## Tags

webassembly, sql, olap, analytics, browser, database, typescript, cpp, apache-arrow, parquet, client-side
