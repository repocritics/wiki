# unjs/consola

> Console logger for Node.js and the browser — pretty output for CLIs and dev
> tools, not a production log pipeline.

[GitHub repo](https://github.com/unjs/consola) ·
[unjs ecosystem](https://unjs.io) ·
[License: MIT](https://github.com/unjs/consola/blob/main/LICENSE)

## Overview

Consola is the logging layer of the unjs ecosystem — the framework-agnostic
JavaScript packages maintained by Pooya Parsa and the Nuxt core team. Started
in 2018 as Nuxt's console wrapper, it became the default logger of the modern
CLI generation: Nuxt, Nitro, and a long tail of scaffolding and build tools
route their terminal output through it. It provides leveled, typed log methods
(`info`, `success`, `warn`, `box`, ...), pluggable reporters, tag scoping,
global `console`/stdout interception, test mocking, and — since v3 —
interactive prompts backed by `@clack/prompts`[^1].

The defining tradeoff: consola optimizes for *developer-facing* output, not
machine-facing logs. The default "fancy" reporter emits ANSI colors, unicode
icons, and aligned formatting; structured JSON output is possible but is a
write-your-own-reporter exercise, and there is no built-in transport, rotation,
or serialization story. It is closer to a CLI presentation library than to
winston or pino, and choosing it for server-side production logging is the most
common misuse.

v3 (April 2023) was a full TypeScript rewrite: ESM-first with CJS compat,
named exports replacing the v2 default export, `chalk` and `dayjs` dropped,
and `consola.prompt()` added[^2]. The package ships with zero runtime
dependencies — even the clack integration is inlined at build time. Its
~7.3k stars understate its reach as a transitive dependency of Nuxt, Nitro,
and hundreds of CLIs. The repo is actively maintained (pushed July 2026),
though releases are sparse — v3.4.2 (March 2025) has been the latest tag for
over a year while `main` moves on.

## Getting Started

```bash
npm i consola
```

```js
// ESM (CJS: const { consola, createConsola } = require("consola"))
import { consola, createConsola } from "consola";

consola.start("Building project...");
consola.warn("Config field `foo` is deprecated");
consola.success("Project built!");
consola.box("Released v1.2.0");

const ok = await consola.prompt("Deploy to production?", {
  type: "confirm", // also: "text" | "select" | "multiselect"
});
```

Bundle-sensitive consumers can import stripped builds — `consola/basic`,
`consola/browser`, or `consola/core` — which drop the fancy reporter and cut
bundle size by up to 80%[^3].

## Architecture / How It Works

The core is a small `Consola` class that turns each call into a `LogObject`
— `{ date, args, type, level, tag }` — and dispatches it to an array of
reporters; the core does no formatting. Three reporters ship built in: fancy
ANSI (default in interactive TTYs), basic plain-text, and a browser reporter
mapping onto `console.*`. Environment detection via unjs/std-env downgrades
fancy → basic in CI and test environments automatically[^4].

Log *types* (`info`, `success`, `fatal`, `ready`, ...) are presets pairing a
numeric level (0 fatal/error → 5 trace, ±999 silent/verbose) with styling;
the instance `level` (default 3, or the `CONSOLA_LEVEL` env var) filters what
reaches reporters. `withTag()` creates child instances inheriting parent
options — including mocks, so `mockTypes()` works on scoped loggers too.

The more invasive machinery: `wrapConsole()` / `wrapStd()` / `wrapAll()`
monkey-patch the global `console` and stdout/stderr to route third-party
output through consola's pipeline (`restore*()` undoes it). `pauseLogs()`
queues logs globally and flushes on `resumeLogs()` — how CLI frameworks keep
output clean around prompts and spinners. `consola/utils` exports the
formatting primitives (`box`, `colors`, `stripAnsi`, `formatTree`) directly.
Dependency-freedom cuts both ways: std-env, clack, and color utilities are
bundled at build time, so fixes to them only ship via a consola release.

## Production Notes

- **Not a production logger.** No transports, rotation, or serialization
  story. For server telemetry write a one-line JSON reporter
  (`{ log: (obj) => console.log(JSON.stringify(obj)) }`) or use pino, and
  keep consola for the CLI surface.
- **Output differs between local and CI.** The std-env fallback silently
  swaps fancy for basic formatting in CI, TTY-less Docker, and test runners;
  snapshot tests on terminal output pass locally and fail in CI unless you
  pin `fancy: true|false` in `createConsola`.
- **Global wrapping is a footgun.** `wrapAll()` mutates process-wide state;
  overlapping wraps, or one left behind after tests, yields double-formatted
  or swallowed output. `restoreAll()` in teardown; never wrap in a library.
- **The `raw` escape hatch.** Objects with `message` or `args` keys collide
  with the `LogObject` shape — `consola.log({ message: "hello" })` prints
  `hello`, not the object. Use `consola.log.raw(obj)` for arbitrary data[^5].
- **Prompt cancellation is surprising.** Ctrl+C resolves the prompt with the
  default value rather than rejecting; pass `{ cancel: "reject" }` if silent
  fallthrough would corrupt state (configurable since v3.3)[^6].
- **v2 → v3 migration** is mostly mechanical (default export → named
  `consola`), but ESM-first packaging broke some CJS setups on old bundlers;
  v2 remains widespread in transitive dependency trees.
- **`CONSOLA_LEVEL` has gaps** — the env var is not read by the browser and
  core builds; set `level` explicitly there.

## When to Use / When Not

**Use when:**
- Building a CLI, scaffolder, or dev server where terminal output quality is
  part of the product.
- You want logs, boxes, and prompts from one dependency-free package instead
  of assembling chalk + ora + inquirer + a logger.
- You need to mock log output in vitest/jest (`mockTypes`) or intercept a
  noisy dependency's `console` calls.
- You are in the Nuxt/unjs ecosystem — it is the native convention.

**Avoid when:**
- You need production server logging: structured JSON, transports, and
  throughput are pino/winston territory.
- You only need colored text — `picocolors` or `kleur` are far smaller.
- You need rich prompt flows (validation, multi-step wizards) — use
  `@clack/prompts` or Inquirer directly.
- You depend on stable output across environments (see CI fallback above)
  and won't pin reporter config.

## Alternatives

- pinojs/pino — use instead for production logging: NDJSON, transports.
- winstonjs/winston — use instead for multi-transport logging (files, HTTP).
- debug-js/debug — use instead for namespaced diagnostics inside libraries.
- bombshell-dev/clack — use directly when prompts are the main event.
- klaussinani/signale — closest historical competitor; unmaintained.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.1.0 | 2018-03 | Initial release, extracted from Nuxt tooling. |
| v2.0.0 | 2018-11 | Reporter-based architecture era; bundled chalk/dayjs. |
| v3.0.0 | 2023-04-11 | TypeScript rewrite, ESM-first, named exports, dropped chalk/dayjs, `consola.prompt()`[^2]. |
| v3.1.0 | 2023-04-18 | `basic`/`core`/`browser` subpath builds, up to 80% smaller bundles[^3]. |
| v3.2.0 | 2023-06-27 | `consola.box`, `consola/utils` subpath, color utilities[^7]. |
| v3.3.0 | 2024-12-19 | `formatTree`, error `cause` printing, configurable prompt cancel strategy[^6]. |
| v3.4.0 | 2025-01-13 | Switched to upstream `@clack/prompts`[^1]. |
| v3.4.2 | 2025-03-18 | Latest tagged release; development continues on `main`. |

## References

[^1]: consola v3.4.0 release notes — upstream `@clack/prompts`. https://github.com/unjs/consola/releases/tag/v3.4.0
[^2]: consola v3.0.0 release notes — TypeScript rewrite, ESM, prompt, migration from v2. https://github.com/unjs/consola/releases/tag/v3.0.0
[^3]: consola v3.1.0 release notes — subpath exports saving up to 80% bundle size. https://github.com/unjs/consola/releases/tag/v3.1.0
[^4]: consola README — custom reporters and std-env-based environment detection. https://github.com/unjs/consola#custom-reporters
[^5]: consola README — raw logging methods. https://github.com/unjs/consola#raw-logging-methods
[^6]: consola v3.3.0 release notes — configurable `cancel` strategy. https://github.com/unjs/consola/releases/tag/v3.3.0
[^7]: consola v3.2.0 release notes — box, utils subpath, color utils. https://github.com/unjs/consola/releases/tag/v3.2.0

## Tags

typescript, nodejs, logging, cli, console, terminal, developer-tools, unjs, zero-dependency, browser, prompts
