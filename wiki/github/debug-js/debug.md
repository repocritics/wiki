# debug-js/debug

> A tiny namespaced logging utility for Node.js and browsers, toggled entirely through the `DEBUG` environment variable.

[GitHub repo](https://github.com/debug-js/debug) ·
[npm](https://www.npmjs.com/package/debug) ·
[License: MIT](https://github.com/debug-js/debug/blob/master/LICENSE)

## Overview

`debug` is a ~200-line logging shim, originally written by TJ Holowaychuk under the `visionmedia` org in 2011 and now maintained under `debug-js`[^1]. It does one thing: it hands you a decorated `console.error` bound to a namespace string, and lets you switch groups of those namespaces on or off at runtime via the `DEBUG` env var (or `localStorage.debug` in browsers). It is modelled on Node.js core's own internal debug technique (`NODE_DEBUG`).

Its significance is out of all proportion to its size. `debug` is one of the most-depended-upon packages in the npm registry — Express, Mocha, socket.io, Koa, body-parser, and thousands of other libraries pull it in as a runtime dependency for their internal tracing. That ubiquity is the defining tension: `debug` is not really "your" logger, it is the logger the whole dependency tree already speaks, which is what makes `DEBUG=*` both wonderfully universal and a firehose of unrelated internals.

It is deliberately **not** a leveled logger. There is no `info`/`warn`/`error` hierarchy, no structured JSON, no transports, no log rotation. Each namespace is a single binary on/off switch. Teams that reach for it as a production logging framework usually outgrow it and move to `pino` or `winston`; teams that use it for what it is — developer-facing trace output during development — rarely need anything else.

## Getting Started

```bash
npm install debug
```

```js
const debug = require('debug')('myapp:server');

debug('booting %o', { port: 3000 });   // printf-style + object inspection
debug('request received');             // "+2ms" diff auto-appended
```

Nothing prints until you enable the namespace:

```bash
DEBUG=myapp:* node app.js       # enable everything under myapp:
DEBUG=* node app.js             # enable everything, including deps
DEBUG=*,-express:* node app.js  # everything except express internals
```

In the browser, enable via `localStorage.debug = 'myapp:*'` and reload; output goes through the console with `%c` colors.

## Architecture / How It Works

The package is three small files stitched together by environment detection:

- **`src/common.js`** — the engine. It holds `createDebug()`, the `enable()`/`disable()`/`enabled()` logic, the namespace→color hash, the millisecond-diff timer, and the wildcard matcher. The `DEBUG` string is parsed once into two lists (`names` and `skips`) of regexes; a namespace is enabled if it matches a name and no skip. `*` compiles to `.*?`.
- **`src/node.js`** — the Node entry point. It wires in TTY color detection (optionally via the `supports-color` peer), `util.inspect()` for the `%o`/`%O`/`%O` formatters, `DEBUG_*` env-var options, and writes to `process.stderr`.
- **`src/browser.js`** — the browser entry point. It uses `localStorage` for persistence, the console's `%c` styling for per-namespace color, and feature-detects "colored" consoles (WebKit/Firefox).

`package.json`'s `browser` field swaps `node.js` for `browser.js` at bundle time, so `require('debug')` resolves differently depending on target. Shared behavior lives in `common.js` and is injected the environment-specific `formatArgs`, `save`, `load`, and `useColors` functions — a hand-rolled dependency injection rather than a class hierarchy.

Two design details matter downstream. First, enablement is resolved when a debug instance is **created**, and `debug.enable(str)` re-applies to instances lazily. Changing `process.env.DEBUG` alone after startup does not retroactively flip already-created loggers — you must call `debug.enable()`. Second, a disabled `debug(...)` call is guarded by an `.enabled` boolean and returns almost immediately, but the **arguments are still evaluated** by the caller before the call — so `debug('state: %o', expensiveSerialize())` still runs `expensiveSerialize()` even when the namespace is off.

## Production Notes

- **It runs in production whether you want it or not.** Because `debug` is a runtime (not dev) dependency of common frameworks, it is loaded in every deployment. The cost of a *disabled* namespace is a cheap boolean check, but the module and its per-namespace color computation are always present.
- **Guard expensive log arguments.** As above, arguments aren't lazy. For hot paths use `if (debug.enabled) debug(...)` so serialization only runs when enabled.
- **`DEBUG=*` is a data-exposure risk.** Turning everything on in production can dump request bodies, tokens, and connection strings that dependencies happen to trace. Scope namespaces deliberately; never blanket-enable on a system handling secrets.
- **Output goes to stderr, not stdout,** by default. Log collectors that only capture stdout will miss it. Per-namespace redirection is possible by overriding `.log` (e.g. `debug.log = console.log.bind(console)`).
- **Colors need help.** In Node, install `supports-color` for a full palette; without it you get a handful of basic colors. In piped child processes, TTY detection fails and colors vanish unless you pass `DEBUG_COLORS=1`.
- **Historical ReDoS.** Versions before 2.6.9 / 3.1.0 carried a regular-expression denial-of-service advisory (CVE-2017-16137)[^2]; anything on a modern 4.x line is unaffected.
- **Supply-chain target.** Its reach makes `debug` a high-value account to compromise. In September 2025 the maintainer's npm credentials were phished and malicious versions of `debug` (alongside `chalk` and other packages) were briefly published before being detected and removed within hours[^3]. Pin versions and use a lockfile with integrity hashes; this is the practical mitigation for any ubiquitous transitive dependency.

## When to Use / When Not

**Use when:**
- You are a library author and want end users to be able to trace your internals without a heavy logging dependency.
- You want zero-config, env-var-toggled developer trace output in Node or the browser.
- You want to selectively silence or surface subsets of a large dependency tree's internal logs.

**Avoid when:**
- You need leveled, structured, or JSON logging for a production observability pipeline — use `pino` or `winston`.
- You need log routing, sampling, rotation, or transports.
- You are logging in a hot path and cannot afford eager argument evaluation without guards.

## Alternatives

- pinojs/pino — use when you need fast, structured JSON logging with real levels and transports for production.
- winstonjs/winston — use when you need configurable transports, formats, and log levels in one framework.
- unjs/consola — use when you want a prettier, leveled console logger with browser and Node support.
- pimterry/loglevel — use when you want a minimal leveled logger for the browser without namespaces.
- Node's built-in `util.debuglog` — use when you want the same `NODE_DEBUG`-style toggle with zero dependencies inside a Node-only project.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2011-11 | Initial release under `visionmedia/debug` (TJ Holowaychuk)[^1]. |
| 2.6.9 | 2017-09 | Backport line; ReDoS fix (CVE-2017-16137)[^2]. |
| 3.0.0 | 2017-07 | Maintenance moved toward the `debug-js` org; formatter and internals cleanup. |
| 3.1.0 | 2017-10 | ReDoS fix on the 3.x line[^2]. |
| 4.0.0 | 2018-09 | Dropped old Node support, dependency and build modernization; current major line. |
| 4.x | 2020–2026 | Ongoing 4.3.x / 4.4.x maintenance releases; repo last active 2026-04[^4]. |

## References

[^1]: debug-js/debug — repository and README. https://github.com/debug-js/debug
[^2]: CVE-2017-16137 — Regular Expression Denial of Service in `debug`, fixed in 2.6.9 and 3.1.0. https://nvd.nist.gov/vuln/detail/CVE-2017-16137
[^3]: GitHub Advisory / npm — September 2025 phishing compromise of maintainer accounts affecting `debug`, `chalk`, and related packages. https://github.com/advisories
[^4]: GitHub API repository metadata for debug-js/debug (stars, forks, last push), fetched 2026-07. https://github.com/debug-js/debug

## Tags

javascript, nodejs, logging, debugging, cli, browser, developer-tools, environment-variables, npm-library, tracing
