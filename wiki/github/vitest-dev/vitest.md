# vitest-dev/vitest

> A test runner that reuses your Vite config and transform pipeline, so app and tests share one build.

[GitHub repo](https://github.com/vitest-dev/vitest) ·
[Official website](https://vitest.dev) ·
[License: MIT](https://github.com/vitest-dev/vitest/blob/main/LICENSE)

## Overview

Vitest is a JavaScript/TypeScript test runner built on top of Vite, first
released in 2021 out of the Vite/Vue ecosystem and now stewarded by VoidZero
Inc. (the company around Vite's author, Evan You)[^1]. Its central premise is
that a test runner should not maintain its own module transform, resolution, and
plugin stack — it should borrow the one the application already uses. If your app
compiles TypeScript, JSX, path aliases, and PostCSS through Vite, your tests get
the identical treatment for free, which removes the class of "works in the app,
breaks in Jest" configuration drift.

The API is deliberately Jest-shaped: `describe`/`it`/`expect`, `vi.mock`,
snapshot testing, and `vi.fn` spies mirror Jest closely enough that many suites
migrate with a find-and-replace of the import line. Assertions are Chai
underneath with a Jest-`expect`-compatible layer bolted on top[^2]. This makes
Vitest easy to adopt but also means its behavior is *Jest-like*, not
Jest-identical — mock hoisting, module registry semantics, and environment
edge cases differ in ways that surface during real migrations.

The defining tension is coupling to Vite. That coupling is the whole value
proposition and also the constraint: Vitest tracks Vite's major versions and its
evolving module-execution internals, so upgrades are bounded by the Vite
release you can adopt. As of this writing the project requires Vite >=6.4 and
Node >=22.12[^3], a fairly aggressive floor for a test tool.

## Getting Started

```bash
npm install -D vitest
# add to package.json: "test": "vitest"
```

```ts
// math.test.ts
import { describe, expect, it, vi } from 'vitest'
import { add } from './math'

describe('add', () => {
  it('sums two numbers', () => {
    expect(add(1, 2)).toBe(3)
  })

  it('can mock a dependency', () => {
    const spy = vi.fn().mockReturnValue(42)
    expect(spy()).toBe(42)
    expect(spy).toHaveBeenCalledOnce()
  })
})
```

```bash
npx vitest          # watch mode by default
npx vitest run      # single pass, for CI
```

Configuration lives in `vitest.config.ts`, or under a `test` key in an existing
`vite.config.ts`. Setting `test.globals: true` restores Jest-style ambient
`describe`/`expect` without imports.

## Architecture / How It Works

Vitest runs a Vite dev server in the background purely as a transform and module
service. Test files and their imports are compiled on demand through Vite's
pipeline (esbuild/SWC for TS/JSX, plugins, resolvers), then executed inside
worker processes. The mechanism that pulls transformed modules into the runner
was originally a package called `vite-node`, and has since moved onto Vite's
own Module Runner / Environment API as those stabilized in Vite 5–6[^4]. This is
why the Vite version floor matters: the runner rides on internal Vite machinery,
not a frozen public contract.

Test isolation is handled by a worker pool (Tinypool). The default pool is
`forks` (child processes via `node:child_process`), chosen over
`threads` because worker threads interact badly with some native addons and
global state[^5]; `threads`, `vmThreads`, and `vmForks` remain selectable for
speed-vs-isolation tradeoffs. Each test file gets a fresh module graph by
default, which is what makes `vi.mock` and module state reset between files.

The DOM is not real. Browser-API tests run against a simulated environment —
`jsdom` (spec-complete, slower) or `happy-dom` (faster, less complete) — selected
per file or globally via `environment`. For genuine browser semantics, **Browser
Mode** drives tests in a real Chromium/Firefox/WebKit instance through Playwright
or WebdriverIO instead of a simulated DOM; it began as experimental and has since
been declared stable[^6].

Other internals worth knowing: coverage comes from either V8's native counters
(fast, occasionally imprecise on branch mapping) or Istanbul instrumentation
(slower, more precise); snapshots and `expect` compatibility are Jest-modeled;
and type-level testing (`assertType`, `expect-type`) runs a separate `tsc`/
`vue-tsc` pass rather than executing anything.

## Production Notes

**Mock hoisting is the top migration footgun.** `vi.mock(path, factory)` is
hoisted to the top of the file, so the factory cannot reference outer-scope
variables unless they are wrapped — Vitest exposes `vi.hoisted()` for values that
must exist before the mock runs. Suites ported from Jest frequently break here
because Jest's hoisting rules are similar but not identical.

**Pool choice affects both speed and correctness.** `forks` (the default) is the
safe option; switching to `threads` for speed can expose bugs in code that
assumes process-global isolation or uses native modules. Test suites that were
green under one pool can fail under another — treat the pool as part of your test
contract, not a free tuning knob.

**ESM-first has CJS interop costs.** Vitest is ESM-native. Dependencies shipped
as CommonJS, or packages with broken `exports` maps, can require
`server.deps.inline` / `deps.optimizer` configuration to be transformed rather
than externalized. This is the single most common source of "cannot use import
statement outside a module" style failures.

**Config coupling cuts both ways.** Sharing one config with the app is the
feature, but a Vite plugin that behaves differently under test (e.g. one that
injects environment-specific code) will silently affect your suite. Teams with
divergent needs split into a dedicated `vitest.config.ts` with `mergeConfig`.

**Upgrades track Vite majors.** Vitest 2.0 and 3.0 each shipped breaking changes
alongside new Vite requirements[^7]; you generally cannot upgrade Vitest far
ahead of the Vite version your app is pinned to. Monorepos with mixed Vite
versions feel this most.

**Watch mode is stateful.** The instant re-run on save is a real productivity
win, but flaky tests that depend on timing or shared external state can behave
differently in watch vs `vitest run`; CI should always use `run`.

## When to Use / When Not

**Use when:**
- Your project already builds with Vite — the config reuse is close to free.
- You want a Jest-compatible API with first-class ESM and TypeScript and no Babel
  transform to configure.
- You need in-source testing, type-level assertions, or real-browser component
  tests in one tool.

**Avoid when:**
- You are not on Vite and don't intend to be — much of the advantage evaporates,
  and Jest or `node:test` carry less coupling.
- You need a rock-stable runner that rarely breaks across years; Vitest's tie to
  Vite internals means a faster major-version cadence.
- Your dependency tree is heavily CommonJS with fragile `exports` maps and you
  don't want to spend time on `deps` configuration.

## Alternatives

- jestjs/jest — the incumbent; use it when you are not on Vite and want the most
  mature ecosystem and plugin/matcher library, accepting slower native ESM.
- nodejs/node (`node:test`) — use when you want zero dependencies and a
  runtime-native runner and can live without Jest's mocking/snapshot surface.
- avajs/ava — use when you want minimal, concurrent-by-default tests with an
  opinionated small API.
- mochajs/mocha — use when you want an unopinionated core and to assemble your own
  assertion/mocking stack.
- oven-sh/bun (`bun test`) — use when you're already on the Bun runtime and want a
  built-in Jest-compatible runner with no separate install.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2021-12 | Initial public releases from the Vite/Vue ecosystem[^1]. |
| 1.0 | 2023-12-04 | First stable major; API and config settled[^7]. |
| 2.0 | 2024-07 | Reworked reporters, `forks` default pool, breaking changes[^5][^7]. |
| 3.0 | 2025-01 | Vite 6 alignment, config and API breaking changes[^7]. |
| 3.2 | 2025 | Browser Mode declared stable[^6]. |

## References

[^1]: Vitest documentation, "Why Vitest?" — project background and Vite
relationship. https://vitest.dev/guide/why
[^2]: Vitest documentation, "Expect" API — Chai + Jest-compatible assertions.
https://vitest.dev/api/expect
[^3]: vitest-dev/vitest README — "Vitest requires Vite >=v6.4.0 and Node
>=v22.12.0". https://github.com/vitest-dev/vitest
[^4]: Vite documentation, "Environment API" / Module Runner. https://vite.dev/guide/api-environment
[^5]: Vitest documentation, "Improving Performance" — pool options and the
`forks` default. https://vitest.dev/guide/improving-performance
[^6]: Vitest documentation, "Browser Mode". https://vitest.dev/guide/browser/
[^7]: Vitest migration guides / release blog. https://vitest.dev/guide/migration
[^8]: License — MIT © 2021-Present VoidZero Inc. and Vitest contributors.
https://github.com/vitest-dev/vitest/blob/main/LICENSE

## Tags

testing, test-runner, javascript, typescript, vite, unit-testing, esm, jest-compatible, browser-testing, coverage, mocking, frontend
