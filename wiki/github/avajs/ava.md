# avajs/ava

> A Node.js test runner that isolates each test file in its own process and runs everything concurrently by default.

[GitHub repo](https://github.com/avajs/ava) ·
[Official website](https://avajs.dev) ·
[License: MIT](https://github.com/avajs/ava/blob/main/license)

## Overview

AVA is a test runner for Node.js, started by Sindre Sorhus in 2015 and now maintained primarily by Mark Wubben[^1]. Its design bets on two ideas that most other JavaScript test frameworks do not enforce: **tests should be atomic** (no shared mutable state, no `describe`/`it` nesting, no implicit globals), and **tests should run concurrently** (each test file in a separate Node.js worker, and tests within a file interleaved on the event loop). The result is a flat, promise-native API — you `import test from 'ava'` and register callbacks that receive an assertion context `t`.

The defining tradeoff is process isolation. AVA forks a worker per test file, which gives clean environment separation (one file's globals, mocks, and module cache cannot leak into another) at the cost of higher per-file startup and memory overhead than in-process runners. This is a deliberate philosophical stance: AVA would rather pay for isolation than let tests share state and interfere with each other. It is a good fit for suites of independent unit tests and a poor fit for suites that want to share expensive setup across many files.

AVA is ESM-first and unopinionated about the rest of the stack. It ships no mocking library, no code-coverage instrumentation, and no assertion DSL beyond its built-in `t.*` methods. You bring `sinon`, `c8`/`nyc`, and any test doubles yourself. The API surface is intentionally small, which is both its appeal (little to learn, fast to read) and its limitation (batteries not included).

## Getting Started

AVA must be installed locally — it cannot be run as a global binary.

```bash
npm init ava        # scaffolds config + test script
# or: npm install --save-dev ava
```

```js
// test.js
import test from 'ava';

test('addition is commutative', t => {
	t.is(1 + 2, 2 + 1);
});

test('resolves the promise', async t => {
	const value = await Promise.resolve('bar');
	t.is(value, 'bar');       // magic assert prints a diff on failure
});

test.serial('runs after previous serial tests', t => {
	t.true(Array.isArray([]));
});
```

```bash
npx ava            # run once
npx ava --watch    # watch mode
```

Configuration lives under an `"ava"` key in `package.json`, or in `ava.config.js` / `ava.config.mjs`.

## Architecture / How It Works

The core execution model is **one Node.js worker process per test file**. The main process discovers test files (glob-based), then schedules them across a pool of workers whose size defaults to the number of available CPU cores. Files run in parallel; the tests *inside* a single file run concurrently on that worker's event loop unless declared with `test.serial`, which forces sequential ordering within the file.

This is why AVA forbids shared state: because concurrent tests interleave, any mutation of a variable shared between two `test()` bodies is a race. AVA leans into this by giving each test its own `t` context object and encouraging per-test setup via `t.context` populated in `test.beforeEach`.

Assertions go through a small fixed set of methods (`t.is`, `t.deepEqual`, `t.throws`, `t.snapshot`, `t.like`, and others). Failure output is AVA's **"magic assert"**: it reads the failing source line, renders the actual value with syntax highlighting, and — for objects, arrays, and multi-line strings — shows a minimized diff rather than dumping both sides. This is powered internally by the `concordance` serializer/diff library.

A notable historical inflection: early AVA (1.x–2.x) bundled Babel and transpiled your tests by default. AVA 3 removed built-in Babel, making transpilation opt-in via the separate `@ava/babel` package[^2]. Subsequent majors moved the project toward native ESM as the documented default, with CommonJS still supported but treated as the legacy path. TypeScript is handled out-of-band by `@ava/typescript`, a loader/config integration — AVA itself does not type-check.

Cross-file coordination, when you truly need it, is provided by opt-in companion packages: `@ava/cooperate` (low-level primitives for inter-file cooperation) and `@ava/get-port` (reserve a port across the whole run). These exist precisely because the default isolation makes sharing hard.

## Production Notes

- **Worker overhead is the main scaling cost.** Because every test file spawns a process, suites with hundreds of tiny files pay repeated Node startup and memory cost. In-process runners (Vitest, Jest with its worker pooling) can be faster for such shapes. Consolidating many small files, or tuning the `concurrency` option, is the usual mitigation.
- **Concurrency is a footgun for stateful tests.** Tests that touch a shared database, filesystem path, or module-level singleton will interleave nondeterministically. The fixes are `test.serial`, per-test isolation via `t.context`, or reserving resources (unique DB schemas, `@ava/get-port`). Migrating a Mocha/Jest suite that assumed sequential execution frequently surfaces latent ordering bugs.
- **No built-in mocking or coverage.** There is no `jest.mock` equivalent and no `--coverage`. Coverage is typically run as `c8 ava`. Module mocking in ESM is genuinely harder here than in Jest and often requires dependency injection or loader tricks.
- **ESM-first can bite CommonJS projects.** The documentation assumes ES modules. CommonJS works, but configuration interplay (`"type": "module"`, file extensions, transpilation) is a recurring source of setup confusion for teams on older codebases.
- **Parallel CI is automatic but env-dependent.** AVA detects supported CI providers (via the `ci-parallel-vars` package) and shards test files across parallel machine jobs. On an unsupported or misconfigured CI it silently runs everything on one node.
- **Global install does not work.** AVA intentionally refuses to run from a global install; the binary must be resolved from the project's `node_modules`.

## When to Use / When Not

**Use when:**
- You want fast, isolated unit tests with no shared global state and no `describe` nesting.
- You value a tiny, promise-native API and readable failure diffs over an all-in-one toolkit.
- Your tests are naturally independent and benefit from running concurrently.
- You are on a modern ESM codebase and comfortable assembling mocking/coverage yourself.

**Avoid when:**
- Your suite depends on expensive shared setup across many files, or on sequential execution.
- You want a batteries-included runner with built-in mocking, coverage, and a jsdom environment (reach for Jest or Vitest).
- You are testing browser/React components and want a Vite-native or jsdom-first workflow.
- You have a large legacy CommonJS suite where AVA's ESM-first defaults add friction.

## Alternatives

- facebook/jest — use when you want a batteries-included runner with mocking, coverage, and jsdom in one package, especially for React.
- vitest-dev/vitest — use in Vite projects, or when you want a Jest-compatible API with fast ESM-native execution.
- nodejs/node (`node:test`) — use when you want zero third-party dependencies and only need the built-in test runner and assertions.
- mochajs/mocha — use when you want maximum flexibility and to compose your own assertion and mocking libraries.
- sinonjs/sinon — not a runner but the common companion for the mocking/stubbing AVA deliberately omits.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2015 | Initial releases; concurrent, atomic-test philosophy established[^1]. |
| 1.0 | 2018-12 | First stable major after a long 0.x line. |
| 2.0 | 2019-06 | Reporter and API refinements. |
| 3.0 | 2020-01 | Built-in Babel removed; transpilation moved to opt-in `@ava/babel`[^2]. |
| 4.0 | 2022-01 | Continued modernization; watcher and config changes. |
| 5.0 | 2022-10 | Node.js support baseline raised; ESM-forward defaults. |
| 6.0 | 2023-12 | Current major line; ESM-first, updated Node.js requirements. |

## References

[^1]: AVA README and repository — avajs/ava. https://github.com/avajs/ava
[^2]: `@ava/babel` — optional Babel support extracted from AVA core in v3. https://github.com/avajs/babel
[^3]: AVA documentation site. https://avajs.dev

## Tags

javascript, nodejs, testing, test-runner, test-framework, unit-testing, tdd, concurrency, esm, assertions, cli
