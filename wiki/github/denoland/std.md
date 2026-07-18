# denoland/std

> The Deno Standard Library — audited, web-standards-first building blocks,
> published as independent `@std/*` packages on JSR.

[GitHub repo](https://github.com/denoland/std) ·
[Official website](https://jsr.io/@std) ·
[License: MIT](https://github.com/denoland/std/blob/main/LICENSE)

## Overview

`denoland/std` (formerly `deno_std`; the old name still redirects) is the
standard library for the Deno runtime, started in November 2018 alongside Deno
itself. It fills the gap between Deno's deliberately small built-in API surface
and what real programs need: paths, filesystem helpers, HTTP utilities,
assertions, CSV/YAML/TOML/JSONC parsing, encoding, async primitives, CLI
argument parsing, semver, UUIDs, streams — each shipped as its own semver'd
package (`@std/path`, `@std/assert`, `@std/csv`, ...).

The library has lived through two identity shifts. Early std imitated Go's
standard library — including Go-style `Reader`/`Writer` I/O interfaces — and
was imported by URL from `deno.land/std@0.x/`, versioned as one monolithic 0.x
release train. Around 2024 both decisions were reversed: the I/O layer was
rebuilt on Web Streams and other web standards[^2], and distribution moved to
JSR with per-package semantic versioning; `deno.land/std` was frozen at
0.224.0[^1]. The first wave of packages reached stable 1.0.0 in 2024[^3].

The 3.5k GitHub stars badly understate reach: std is a default dependency of
most non-trivial Deno code, and JSR's `@std` scope is among the most-downloaded
on the registry. Maintenance is healthy — pushes within days of writing and
~300 open issues under active triage by paid Deno company staff, not
volunteers.

## Getting Started

```bash
# Deno
deno add jsr:@std/assert jsr:@std/path

# Node.js / Bun (via JSR's npm compatibility layer)
npx jsr add @std/path
```

```ts
import { assertEquals } from "@std/assert";
import { join } from "@std/path";
import { delay } from "@std/async";

Deno.test("join builds platform-correct paths", async () => {
  await delay(10);
  assertEquals(join("users", "brad"), "users/brad");
});
```

Run with `deno test`. On Node/Bun, packages that only use web-standard APIs
work as normal npm imports.

## Architecture / How It Works

The repo is a monorepo where each top-level directory is one independently
versioned JSR package with its own `deno.json`. Design rules are written down
in an in-repo architecture guide[^2] and enforced in review:

- **Web standards first.** APIs are built on `ReadableStream`, `Uint8Array`,
  `URL`, `AbortSignal` — not runtime-specific primitives. The Go-derived
  `Reader`/`Writer` interfaces of early std were removed in the pre-1.0
  cleanup.
- **Fine-grained modules.** Most functions live in single-export files
  (`@std/path/join`, `@std/collections/chunk`), so bundlers tree-shake well and
  you can import one function without pulling a kitchen sink.
- **Minimal coupling.** Inter-package dependencies are kept few and explicit;
  there is no shared "internal utils" grab bag exposed to users.
- **Stability tiers.** Packages at >=1.0.0 follow strict semver; sub-1.0
  packages (e.g. `@std/log`, `@std/datetime` for long stretches) follow a
  documented 0.x breaking-change convention[^4] and can churn between minors.
- **Runtime portability is per-package.** JSR displays compatibility per
  package. Anything touching the `Deno.*` namespace is Deno-only;
  pure-web-standard packages run on Node, Bun, browsers, and edge runtimes.

Test coverage is tracked publicly per package[^5]; stable packages require
near-complete coverage plus documented examples on every exported symbol.

## Production Notes

- **The URL-import era is over.** `deno.land/std` serves nothing newer than
  0.224.0[^1]. Old URLs keep resolving, but fixes land only on JSR; codebases
  on `https://deno.land/std@0.x/...` imports face a mechanical but tedious
  migration to `jsr:@std/*` specifiers.
- **Pre-1.0 history means old tutorials lie.** Years of 0.x breaking changes:
  `std/http`'s `serve()` removed in favor of built-in `Deno.serve`; `std/node`
  moved into the Deno CLI; `std/flags` became `@std/cli`'s `parseArgs`;
  `std/testing/asserts` became `@std/assert`. Assume any std snippet from
  before 2024 is stale until checked against JSR docs.
- **Mixed versions are normal, pin them anyway.** Per-package semver means one
  project legitimately holds `@std/path@1.x` next to `@std/log@0.x`. Commit
  the lockfile; sub-1.0 packages can break on minor bumps.
- **Node/Bun usage works but read the compat badge.** JSR transpiles `@std/*`
  for npm consumers, and web-standard packages behave identically. Packages
  that reach for `Deno.*` fail outside Deno — and the failure is late (call
  time), not at install time.
- **Don't reimplement what the runtime absorbed.** Deno has steadily pulled std
  functionality into the CLI (serve, Node compat). Check whether a built-in
  exists before adding the std package; the std team itself deprecates in that
  direction.

## When to Use / When Not

**Use when:**

- You write Deno code at all — std is the sanctioned first stop before any
  third-party dependency.
- You want coverage-tested parsing (CSV, YAML, TOML, JSONC), encoding, or path
  handling instead of picking among npm micro-packages.
- You need cross-runtime utilities built on web standards that behave the same
  in Deno, Node, Bun, and edge workers.

**Avoid when:**

- You target Node exclusively — Node's own core modules plus established npm
  libraries have deeper ecosystem support, and JSR adds an indirection layer.
- You need a package still at 0.x and can't absorb breaking minor releases.
- You need what std deliberately excludes (ORMs, frameworks, heavy date math).

## Alternatives

- nodejs/node — when targeting Node only, its built-in `node:` core modules
  cover most of the same ground with zero dependencies.
- es-toolkit/es-toolkit — broader general-purpose utility belt (lodash
  successor) when you need collection/function helpers beyond std's scope.
- date-fns/date-fns — full-featured date manipulation; `@std/datetime` is thin
  and long sat below 1.0.
- unjs/pathe — drop-in cross-runtime path utilities when you want npm-native
  distribution rather than JSR.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2018-11 | Started alongside pre-1.0 Deno; Go-stdlib-inspired design. |
| 0.x train | 2019–2024 | Monolithic versioning, URL imports from deno.land/std, frequent breaking changes. |
| JSR migration | 2024 | Split into per-package `@std/*` on JSR, independent semver[^3]. |
| 0.224.0 | 2024 | Final release published to deno.land/std; URL distribution frozen[^1]. |
| @std 1.0.0 | 2024 | First wave of packages stabilized under strict semver[^3]. |

## References

[^1]: denoland/std README — "Newer versions ... hosted on JSR. Older versions up till 0.224.0 are still available at deno.land/std." https://github.com/denoland/std
[^2]: Deno Standard Library architecture guide. https://github.com/denoland/std/blob/main/.github/ARCHITECTURE.md
[^3]: JSR `@std` scope — per-package versions and stability status. https://jsr.io/@std
[^4]: Pre-1.0 versioning follows the semver <1.0.0 proposal. https://github.com/semver/semver/pull/923
[^5]: Deno std test coverage explorer. https://std-coverage.deno.dev/

## Tags

typescript, deno, standard-library, jsr, javascript, utilities, web-standards, cross-runtime, monorepo, testing
