# sinonjs/sinon

> Standalone, framework-agnostic test spies, stubs, and mocks for JavaScript.

[GitHub repo](https://github.com/sinonjs/sinon) ·
[Official website](https://sinonjs.org/) ·
[License: BSD-3-Clause](https://github.com/sinonjs/sinon/blob/main/LICENSE)

## Overview

Sinon.JS provides test doubles — spies, stubs, mocks, fakes, and fake timers —
that work with any test runner. It was created by Christian Johansen alongside
his book *Test-Driven JavaScript Development* (2010), and the repository dates to
2010[^1]. Its selling point has always been independence: unlike a mocking API
bundled into one framework, Sinon slots into Mocha, Jasmine, QUnit, `node:test`,
Vitest, or a bare script equally well.

The library draws a deliberate distinction between the doubles it offers. A
**spy** wraps a function and records how it was called without changing behavior.
A **stub** is a spy that also replaces behavior — return values, thrown errors,
resolved/rejected promises, per-argument responses. A **mock** goes further and
bakes *expectations* in up front, failing at `verify()` time if they were not
met. Newer `sinon.fake` and `sinon.replace` APIs cover most stubbing needs with
a smaller surface. Understanding which tool fits which assertion is most of the
learning curve.

The defining tension is between Sinon's flexibility and the discipline it
demands. It can replace almost any property on almost any object, which makes it
possible to reach into code that was never designed for testing — and equally
easy to leave global state mutated between tests. Sinon's answer is the
**sandbox**, but using it correctly is a convention the library cannot enforce.

## Getting Started

```bash
npm install --save-dev sinon
```

```js
const sinon = require("sinon");
const assert = require("node:assert");

// A stub that replaces a method and programs its behavior.
const db = { save: () => {} };
const save = sinon.stub(db, "save").resolves({ id: 1 });

await db.save({ name: "Tom" });

assert.ok(save.calledOnce);
assert.deepEqual(save.firstCall.args[0], { name: "Tom" });

save.restore();        // undo the replacement
```

Prefer a sandbox so every double is cleaned up in one call:

```js
const sandbox = sinon.createSandbox();

afterEach(() => sandbox.restore());   // restores stubs, spies, fake timers

it("charges once", () => {
  const charge = sandbox.stub(payments, "charge").returns(true);
  checkout(cart);
  sinon.assert.calledOnce(charge);
});
```

## Architecture / How It Works

Sinon is a facade over several smaller single-purpose packages under the
`@sinonjs` organization, extracted over years of refactoring[^2]:

- **`@sinonjs/fake-timers`** (formerly `lolex`) — the fake clock. `useFakeTimers`
  installs replacements for `setTimeout`, `setInterval`, `Date`, and (optionally)
  `requestAnimationFrame`, `queueMicrotask`, and `process.hrtime`. `clock.tick()`
  advances virtual time synchronously.
- **`nise`** — fake `XMLHttpRequest` and the fake server (`sinon.fakeServer`) used
  to intercept browser XHR without a network.
- **`@sinonjs/samsam`** — deep equality and the `sinon.match` matcher engine.
- **`@sinonjs/commons`** and **`@sinonjs/formatio`** — shared utilities and the
  value formatting used in assertion messages.

A **spy** is a wrapper function that appends a call record (args, `this`, return
value, exception, call order) to a list on each invocation; the `calledWith`,
`calledOnce`, `returned`, and `threw` predicates query that list. A **stub**
extends a spy with a behavior chain (`returns`, `throws`, `resolves`, `rejects`,
`callsFake`, `withArgs`) evaluated top-down per call. `sinon.assert` reads the
same call records and throws framework-neutral `AssertError`s with formatted
diffs on failure.

The **sandbox** is the coupling story. `sinon.stub`/`sinon.spy` at the top level
mutate real objects and must be individually `restore()`d; a sandbox tracks every
double it creates so a single `restore()` reverts all of them, resets history,
and uninstalls fake timers. The default export `sinon` is itself a global
sandbox, which is convenient and a leak source in equal measure.

## Production Notes

**Leaked doubles are the classic footgun.** A top-level `sinon.stub(obj, "m")`
with no `restore()` — or an assertion that throws before `restore()` runs —
leaves the replacement installed, so later tests see a stubbed method and fail in
confusing, order-dependent ways. Always stub through a sandbox and `restore()` in
an `afterEach` (or `sinon.restore()` for the default sandbox), unconditionally.

**ESM named exports cannot be stubbed.** ES module bindings are immutable by
spec, so `sinon.stub(esmModule, "namedExport")` throws or silently no-ops
depending on the loader. This is not a Sinon bug; it applies to every mocking
library. Workarounds are dependency injection, importing the namespace object and
stubbing a property of it where the bundler allows, or a loader-level mock
(`esmock`, Vitest's `vi.mock`). Do not assume CommonJS stubbing habits carry over.

**Fake timers replace globals wholesale.** After `useFakeTimers`, nothing that
relies on real time advances until you `tick()` — including promise-based delays,
libraries doing their own `setTimeout`, and `Date.now()`. Async code awaiting a
faked timer needs `await clock.tickAsync()` (or `runAllAsync`) so microtasks
flush between timers. Forgetting to restore the clock hangs unrelated tests.

**Sinon vs. framework-native doubles.** Jest (`jest.fn`/`jest.mock`) and Vitest
(`vi.fn`/`vi.mock`) ship their own spies plus module-level auto-mocking that Sinon
does not attempt — Sinon replaces object properties, not module resolution. In a
Jest or Vitest project the native tools are usually the lower-friction choice;
Sinon shines with Mocha, `node:test`, browser suites, and code that needs its
richer behavior chains or standalone fake server.

**`calledWith` uses deep matching.** Assertions compare arguments with samsam's
deep equality, not `===`. For partial or fuzzy matching use `sinon.match(...)`
rather than reconstructing exact argument objects, which are brittle.

## When to Use / When Not

**Use when:**
- Your runner has no built-in mocking (Mocha, QUnit, `node:test`, plain browser).
- You need programmable stub behavior: per-argument returns, promise resolution,
  sequenced call responses, or callback invocation.
- You need deterministic time (`fake-timers`) or a fake XHR/server without a
  network.
- You want assertions and doubles that are portable across test frameworks.

**Avoid when:**
- You already use Jest or Vitest and only need basic spies/mocks — their native
  APIs and module auto-mocking cover it with less setup.
- Your codebase is ESM-first and you need module-level replacement — reach for a
  loader-based mock instead.
- You want zero discipline around teardown — Sinon's power to mutate globals
  punishes teams that skip sandbox restoration.

## Alternatives

- testdouble/testdouble.js — opinionated, "less is more" doubles; use when you
  want a stricter, smaller API that discourages over-mocking.
- jestjs/jest — use its built-in `jest.fn`/`jest.mock` when Jest is already your
  runner and you want module auto-mocking out of the box.
- vitest-dev/vitest — use `vi.fn`/`vi.mock` when on Vite/Vitest; native ESM
  mocking that Sinon cannot do.
- tinylibs/tinyspy — minimal spy library (Vitest's underpinning); use when you
  only need call recording with no stubbing or timers.
- Node's built-in `node:test` `mock` — use for zero-dependency spying/timer
  faking on modern Node when you don't need Sinon's behavior chains.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2010-06 | Christian Johansen; companion to *TDD JavaScript Development*[^1]. |
| 2.0 | 2017-05 | Major API cleanup; async assertion improvements. |
| 4.0 | 2017-09 | `sinon.createStubInstance`, behavior refinements. |
| 6.0 | 2018-06 | Default sandbox becomes the top-level `sinon` export. |
| 7.0 | 2018-10 | `nise`/`fake-timers` extracted to `@sinonjs`[^2]. |
| 9.0 | 2020-02 | Node 10+ baseline; internal modernization. |
| 14.0 | 2022-05 | Drops legacy Node; `@sinonjs` package bumps. |
| 15.0 | 2022-11 | ESM/exports map maintenance. |
| 18.0 | 2024-05 | Node 18+ baseline. |
| 20.0 | 2025-03 | Continued platform-support updates. |
| 22.0 | 2026-05 | Latest major line[^3]. |

## References

[^1]: Sinon.JS homepage and project history. https://sinonjs.org/
[^2]: `@sinonjs` organization — extracted packages (`fake-timers`, `nise`, `samsam`, `commons`). https://github.com/sinonjs
[^3]: Sinon.JS releases and changelog. https://github.com/sinonjs/sinon/releases

## Tags

javascript, testing, test-doubles, mocking, spies, stubs, mocks, fake-timers, unit-testing, tdd, nodejs, test-framework-agnostic
