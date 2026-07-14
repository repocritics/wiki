# mochajs/mocha

> The unopinionated JavaScript test framework: it runs your tests and reports results, and leaves assertions, mocking, and almost everything else to you.

[GitHub repo](https://github.com/mochajs/mocha) ·
[Official website](https://mochajs.org) ·
[License: MIT](https://github.com/mochajs/mocha/blob/main/LICENSE)

## Overview

Mocha is a test framework for Node.js and browsers, first released in 2011 by TJ Holowaychuk[^1]. It predates Jest by five years and, for much of the 2010s, was the default choice for testing JavaScript. Its defining design decision is subtraction: Mocha provides the test *runner* — the `describe`/`it` structure, hooks, timeouts, reporters, and the CLI — but ships no assertion library, no mocking, no spies, and no snapshot support. You are expected to bring your own (`chai`, Node's `assert`, `sinon`, etc.). This is the framework's central tension: it is flexible and composable, but "set up a test stack" is a multi-package decision rather than one `npm install`.

Today Mocha is an independent, volunteer-run project under the OpenJS Foundation[^2] rather than a corporate-backed one, which shows in its cadence: it is maintained and still one of the most-depended-upon packages on npm, but new features arrive slowly and the momentum in the wider ecosystem has shifted to batteries-included runners (Jest, Vitest) and to Node's own built-in `node:test`. With ~23k stars and heavy transitive dependence, Mocha's role in 2026 is less "the runner you pick for a greenfield project" and more "the runner a very large amount of existing code already runs on."

It remains a reasonable, deliberate choice when you want a small, stable runner you fully control, and when the absence of magic is a feature rather than a gap.

## Getting Started

```bash
npm install --save-dev mocha chai
```

```js
// test/math.test.js
import { expect } from "chai";

function add(a, b) { return a + b; }

describe("add()", function () {
  it("sums two numbers", function () {
    expect(add(2, 3)).to.equal(5);
  });

  it("supports async", async function () {
    const v = await Promise.resolve(42);
    expect(v).to.equal(42);
  });
});
```

```bash
npx mocha                    # runs ./test/*.spec.js by default
npx mocha --parallel         # run test files across worker processes
npx mocha --watch            # re-run on file change
```

Configuration lives in `.mocharc.json` / `.mocharc.js` / `.mocharc.cjs` (spec glob, `require` hooks, timeout, reporter).

## Architecture / How It Works

Mocha's runtime is a small set of coupled pieces:

- **Interfaces** define the global vocabulary. The default is BDD (`describe`, `it`, `before`, `beforeEach`), but Mocha also ships TDD (`suite`/`test`), `exports`, and `qunit` interfaces. The interface is a thin layer that registers suites and tests onto an internal tree[^3].
- **Suite / Test tree.** Every `describe` builds a `Suite`; every `it` builds a `Test`. Hooks attach to suites. The `Runner` walks this tree depth-first, executing hooks in the documented order (`before` → `beforeEach` → test → `afterEach` → ... → `after`) and emitting events (`test`, `pass`, `fail`, `end`).
- **Reporters** are event listeners on the `Runner`. `spec`, `dot`, `tap`, `json`, `min`, and others are all just subscribers; custom reporters are ordinary classes. This event-stream design is why third-party reporters (mochawesome, mocha-junit-reporter) are trivial to write.
- **Async model.** A test is async if it (a) declares a `done` callback parameter, or (b) returns a Promise. Mixing the two — taking `done` *and* returning a Promise — is an error ("resolution method is overspecified"). `async`/`await` is the modern path and returns a Promise implicitly.

The most consequential internal detail is that hooks and tests run with a Mocha-managed `this` context (`this.timeout()`, `this.skip()`, `this.retries()`, `this.slow()`). Because arrow functions capture lexical `this`, using them for `it`/`describe` bodies silently breaks all of that. This is not a bug but a direct consequence of the context design, and it is the single most common source of confusion for new users.

Parallel mode (added in v8) forks worker processes via a worker pool and distributes *files*, not individual tests[^4]. Workers do not share module state, so global setup must be expressed through **root hook plugins** and **global fixtures** rather than top-level side effects.

## Production Notes

- **The default timeout is 2000 ms.** On slow or contended CI this produces intermittent, misleading failures that look like logic bugs. Set a realistic per-suite timeout (`this.timeout(...)` or `--timeout`) rather than chasing flakiness.
- **`--exit` and hanging processes.** Since v4, Mocha no longer force-kills the process after tests finish[^5]; if something leaves an open handle (a DB pool, a timer, a socket), the run appears to hang after all tests pass. `--exit` masks this, but the honest fix is to close the handle. Treat a hang-after-green as a resource leak, not a Mocha quirk.
- **Parallel mode is not free and not universal.** It only pays off for large suites (worker startup has overhead), it changes reporter behavior (some reporters don't support it), it is incompatible with `--file`, and it forbids assumptions about cross-file ordering. Suites written against a single-process model can fail subtly when parallelized.
- **No isolation between tests by default.** Unlike Jest, Mocha does not reset module registries or globals between tests. Shared mutable state leaks across `it` blocks unless you clean up in `afterEach`. This is more control and more responsibility.
- **ESM and TypeScript need wiring.** ESM works but is sensitive to `type: "module"`, file extensions, and loader flags; TypeScript typically means a `--require ts-node/register` (or a loader) plus matching `tsconfig`. There is no zero-config TS path the way Vitest offers.
- **Assertion-library coupling is yours to own.** Because Mocha reports whatever error an assertion throws, diff quality, negation, and async matchers all come from your assertion library, not Mocha. Upgrades to `chai` (notably its move to ESM-only in recent majors) can break a Mocha setup even though Mocha itself is unchanged.

## When to Use / When Not

**Use when:**
- You want a small, stable runner and prefer assembling your own assertion/mock stack.
- You value explicitness and no hidden module-reset/auto-mock magic.
- You are maintaining an existing large Mocha suite — migration cost rarely justifies switching.
- You test in real browsers and want a runner with a long browser-support history.

**Avoid when:**
- You want batteries included (assertions, mocking, snapshots, coverage) out of the box — reach for Jest or Vitest.
- You are on a Vite/TypeScript/ESM-first stack and want zero-config speed — Vitest fits better.
- You want per-test isolation and parallelism as defaults rather than opt-ins — Ava or Jest.
- You want zero third-party dependencies — Node's built-in `node:test` now covers the basics.

## Alternatives

- jestjs/jest — batteries-included: assertions, mocking, snapshots, parallel-by-default. Use instead when you want one install and strong React/Babel integration.
- vitest-dev/vitest — Vite-native, fast, Jest-compatible API, first-class ESM/TS. Use instead on modern Vite/TypeScript projects.
- avajs/ava — minimal, concurrent, isolated processes per file. Use instead when parallel isolation is the priority.
- nodejs/node (`node:test`) — built into Node, zero dependencies. Use instead when you want no external test-runner dependency at all.
- jasmine/jasmine — older BDD runner with built-in assertions. Use instead when you want Mocha's structure but bundled matchers.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2011-2012 | Initial releases by TJ Holowaychuk; BDD interface, reporters[^1]. |
| 3.0 | 2016-07 | Dropped legacy Node; reporter and CLI changes. |
| 4.0 | 2017-10 | Stopped force-exiting the process after the run[^5]. |
| 6.0 | 2019-02 | `.mocharc` config files, revamped CLI. |
| 8.0 | 2020-07 | Parallel mode, root hook plugins, global fixtures[^4]. |
| 9.0 | 2021-07 | Dropped older Node, improved ESM handling. |
| 10.0 | 2022-05 | Dropped Node 12; maintenance-focused release. |
| 11.0 | 2024 | Node-version support baseline raised. |
| 12.0 | 2025 | Latest major line; continued Node baseline and dependency updates. |

## References

[^1]: Mocha origin and authorship (TJ Holowaychuk). https://github.com/mochajs/mocha/blob/main/CHANGELOG.md
[^2]: Mocha is an independent, volunteer-maintained OpenJS Foundation project. https://mochajs.org/#backers and repository README "Development" section.
[^3]: Mocha interfaces (BDD/TDD/exports/QUnit) documentation. https://mochajs.org/#interfaces
[^4]: Parallel tests, root hook plugins, and global fixtures. https://mochajs.org/#parallel-tests
[^5]: Mocha v4 release: no longer forcibly exits after tests complete. https://github.com/mochajs/mocha/blob/main/CHANGELOG.md
[^6]: Repository metadata (stars, forks, license, activity) via GitHub API, fetched 2026-07-15. https://github.com/mochajs/mocha

## Tags

javascript, testing, test-framework, test-runner, bdd, tdd, nodejs, browser, mocha, unit-testing
