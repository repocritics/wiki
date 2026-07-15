# blakeembrey/change-case

> A small library for converting strings between `camelCase`, `PascalCase`, `snake_case`, `kebab-case`, `CONSTANT_CASE`, and other programmer casings.

[GitHub repo](https://github.com/blakeembrey/change-case) ·
[License: MIT](https://github.com/blakeembrey/change-case/blob/main/LICENSE)

## Overview

`change-case` does one narrow thing: it re-cases identifier-shaped strings. Given
`"TEST_VALUE"` it returns `"testValue"` for camel case, `"two-words"` for kebab,
`"TWO_WORDS"` for constant, and so on. It is the string utility that code
generators, ORMs, config loaders, and API-shape mappers reach for when they need
to bridge naming conventions between a database column, a JSON key, and a
JavaScript field. At ~2.4k stars and 105 forks[^1] it is not a headline project,
but it is a deep dependency: it sits many layers down in tooling that most
JavaScript developers never see directly.

The defining tension of the library is scope discipline versus expectation. Its
documentation is explicit that it is *not* a title-case or human-text formatter —
it "assumes you are switching between different programming cases" and discards
punctuation, so `"WOW! That's crazy."` becomes `wowThatSCrazy` in camel case[^2].
Users who reach for it as a prose formatter get surprising output; that is by
design, not a bug. For genuine title/sentence formatting the same repo ships a
separate `title-case` package.

The other structural fact worth knowing up front: since version 5 the project is
a **pure ESM** monorepo[^2][^3]. This is a hard constraint, not a style choice —
it cannot be `require()`'d and cannot be used with legacy CommonJS module
resolution in TypeScript. That single decision is the source of most of the
friction users hit in 2024–2026 (see Production Notes).

## Getting Started

```bash
npm install change-case
```

```js
import * as changeCase from "change-case";

changeCase.camelCase("TEST_VALUE");    //=> "testValue"
changeCase.snakeCase("fooBar");        //=> "foo_bar"
changeCase.kebabCase("fooBar");        //=> "foo-bar"
changeCase.constantCase("fooBar");     //=> "FOO_BAR"
changeCase.pascalCase("foo bar");      //=> "FooBar"

// Transform object keys recursively:
import * as changeKeys from "change-case/keys";
changeKeys.camelCase({ TEST_KEY: true }); //=> { testKey: true }
```

Built-in methods in the core package: `camelCase`, `capitalCase`,
`constantCase`, `dotCase`, `kebabCase`, `noCase`, `pascalCase`,
`pascalSnakeCase`, `pathCase`, `sentenceCase`, `snakeCase`, `trainCase`[^2].
The sibling packages `sponge-case`, `swap-case`, and `title-case` are published
separately from the same monorepo[^3].

## Architecture / How It Works

Every case function is a thin composition over one shared pipeline: **split into
words, then re-join with a delimiter and per-word casing.** The exported `split`
utility is the real engine — it takes `"fooBar"` or `"FOO_BAR"` and returns
`["foo", "Bar"]` / `["FOO", "BARS"]`-style word arrays by detecting boundaries at
existing delimiters and at lower→upper camel transitions. Each named function
(`camelCase`, `snakeCase`, …) is just `split` plus a join strategy, which is why
the library exposes `split` publicly: you can build your own case function
without forking anything[^2].

Options are passed as a second argument and are consistent across methods:
`delimiter` (the joining character), `split` (override the word splitter),
`locale` (locale-aware upper/lower, or `false` to disable), and
`prefixCharacters` / `suffixCharacters` to retain leading/trailing symbols — e.g.
passing `prefixCharacters: "_"` preserves the underscores in `__typename`[^2].
There is also `mergeAmbiguousCharacters`: by default `pascalCase` and `snakeCase`
separate ambiguous digit boundaries so `V1.2` becomes `V1_2` rather than `V12`,
and setting the flag merges them[^2].

`change-case/keys` is a separate entry point that walks an object and applies a
core method to its keys, with a configurable `depth` (default `1`, i.e. shallow)
so you control how far into nested structures the rename recurses[^2].

The whole thing is written in TypeScript, ships type definitions, and has no
runtime dependencies in the core package — the algorithm is regex-driven string
manipulation, not a parser.

## Production Notes

- **Pure ESM is the number-one integration issue.** Since v5 the package cannot
  be `require()`'d and will not resolve under TypeScript's `node`/CommonJS module
  resolution[^2]. Projects still on CommonJS must either stay on the v4 line, use
  a dynamic `import()`, or move their `tsconfig` to `"moduleResolution":
  "bundler"`/`"nodenext"` with an ESM-compatible target. This trips up a lot of
  Jest-and-ts-node setups. If you cannot migrate, pinning `change-case@^4` is a
  legitimate long-term choice — v4 is stable and the algorithm rarely changes.

- **v4 → v5 is not a drop-in upgrade.** Before v5 the ecosystem was many small
  npm packages (`camel-case`, `snake-case`, `param-case`, `constant-case`, …)
  each depending on a shared `no-case` core. v5 consolidated everything into the
  single `change-case` package with named exports and renamed `paramCase` to
  `kebabCase`. Codebases importing the old individual packages need import
  rewrites, and the split-package versions are effectively frozen[^3].

- **It is a formatter, not a linter.** Output is lossy: punctuation, emoji, and
  non-identifier characters are dropped, and round-tripping is not guaranteed
  (`snakeCase(camelCase(x))` does not always equal a normalized `x`). Do not use
  it to sanitize untrusted input or as a security boundary.

- **Digit and acronym boundaries are opinion, not standard.** Whether `IOStream`,
  `HTTPServer`, or `v2Point0` split the way you expect depends on the ambiguous-
  character and split rules. When your naming convention disagrees, override the
  `split` option rather than post-processing the output.

- **Maintenance is quiet.** The last push was mid-2025[^1] with a small open-issue
  count. The library is mature and largely feature-complete rather than
  abandoned, but do not expect rapid response to feature requests; treat the
  current behavior as the contract.

## When to Use / When Not

**Use when:**
- You need to bridge naming conventions in codegen, ORMs, or API mappers.
- You want a single small dependency covering many casings with a shared option
  surface.
- Your project is already ESM (or can be), and you want typed, zero-dependency
  transforms.

**Avoid when:**
- You are stuck on CommonJS and cannot migrate — stay on `change-case@4` or pick
  a dual-published alternative.
- You need human-readable title/sentence case with punctuation preserved — use
  the sibling `title-case` package instead.
- You only ever need one casing (e.g. just camelCase) — a single-purpose module
  is smaller and avoids the ESM friction.
- You already depend on a utility toolkit that provides these functions.

## Alternatives

- sindresorhus/camelcase — single-purpose, dedicated camelCase (with sibling
  `snake-case`/etc. modules); use when you need exactly one casing and want the
  smallest footprint.
- lodash/lodash — `_.camelCase`, `_.snakeCase`, `_.kebabCase`, `_.startCase`
  cover the common cases; use when lodash is already a dependency and you don't
  want another package.
- unjs/scule — string case utilities from the UnJS ecosystem that ship both CJS
  and ESM; use when you need CommonJS compatibility today.
- angus-c/just — `just-camel-case`, `just-kebab-case`, etc. are zero-dependency
  single-function modules; use when you want no transitive deps and tree-shakable
  granularity.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-04-15 | Repository created; began as the `change-case` string utility[^1]. |
| 4.x | ~2020 | Split-package era: `camel-case`, `snake-case`, `param-case`, etc. over a shared `no-case` core; dual CJS/ESM. |
| 5.0 | 2023 | Monorepo consolidation into a single `change-case` package with named exports; pure ESM; `paramCase` → `kebabCase`; `change-case/keys` entry point[^3]. |
| 5.x | 2025-06 | Latest activity on the `main` branch[^1]. |

## References

[^1]: GitHub API, `repos/blakeembrey/change-case` — 2,406 stars, 105 forks, MIT, created 2013-04-15, last push 2025-06-17 (fetched 2026-07). https://github.com/blakeembrey/change-case
[^2]: change-case package README (core methods, options, `split`, `change-case/keys`). https://github.com/blakeembrey/change-case/blob/main/packages/change-case/README.md
[^3]: change-case monorepo README (package list, pure-ESM note). https://github.com/blakeembrey/change-case

## Tags

typescript, javascript, string-manipulation, case-conversion, camelcase, snake-case, kebab-case, esm, utility, npm-package
