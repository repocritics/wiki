# lodash/lodash

> A JavaScript utility library of ~200 functions for arrays, objects, and collections — the most-depended-on package in the npm ecosystem, and increasingly one you no longer need.

[GitHub repo](https://github.com/lodash/lodash) ·
[Official website](https://lodash.com/) ·
[License: MIT](https://github.com/lodash/lodash/blob/main/LICENSE)

## Overview

Lodash began in 2012 as a fork of Underscore.js by John-David Dalton, aiming for
consistent cross-environment behavior, better performance, and modular builds[^1].
It grew into a near-universal dependency: for years it sat at or near the top of
npm by download volume, pulled in transitively by a large fraction of the
JavaScript ecosystem. Its surface is a flat namespace of small, composable helpers
— `_.map`, `_.groupBy`, `_.debounce`, `_.cloneDeep`, `_.get` — plus a chaining API
and a functional-programming variant (`lodash/fp`).

The defining tension in 2026 is that the language caught up. Much of what made
lodash indispensable in the ES5 era — `_.map`, `_.filter`, `_.find`, `_.includes`,
`_.assign`, `_.flatten` — now has native equivalents (`Array.prototype.flat`/
`flatMap`, `Object.entries`, optional chaining `?.`, nullish coalescing `??`,
structuredClone). Lodash still wins on a smaller set of genuinely-missing
primitives (`debounce`/`throttle`, deep `merge`/`get`/`set` with dynamic paths,
`groupBy` where not yet available, iteratee shorthands), but importing the whole
library for those is now widely considered a mistake.

The project's own status shifted in 2025: the OpenJS Foundation and the Sovereign
Tech Agency announced funding to move lodash to a "Feature-Complete" maturity stage
with a rebooted Technical Steering Committee[^2]. Read that as: kept stable, secure,
and maintained — not expanded. New utility needs are expected to be met by the
language or by newer libraries, not by lodash growth.

## Getting Started

```shell
npm i --save lodash        # CommonJS / bundlers
npm i --save lodash-es     # ESM build, tree-shakeable
```

```js
// Prefer per-method or lodash-es imports so bundlers can drop the rest.
import debounce from "lodash-es/debounce";
import get from "lodash-es/get";

const onResize = debounce(() => layout(), 150);

// Safe dynamic path access with a default.
const city = get(user, "address.city", "unknown");
```

```js
// The classic chaining/lazy API (full build only).
import _ from "lodash";

const result = _(users)
  .filter(u => u.active)
  .map("name")     // iteratee shorthand: _.property("name")
  .take(3)
  .value();        // nothing runs until .value()
```

## Architecture / How It Works

Lodash is authored as a large single source (`lodash.js`) and post-processed by
`lodash-cli` into the shipped artifacts: a UMD full build (~24 kB gzipped), a
trimmed "core" build (~4 kB gzipped), per-method packages, an ES-modules build
(`lodash-es`), an AMD build, and the FP variant[^3]. There is no bundler magic in
the source itself; the modular npm packages are generated, not hand-split.

Three internal ideas do most of the work:

- **Iteratee shorthands.** Most collection methods accept a function, a string
  (`_.property`), an array `[key, value]` (`_.matchesProperty`), or an object
  (`_.matches`). This is why `_.map(users, "name")` works. It is convenient and a
  frequent source of confusion when a value that "looks like" a shorthand is passed
  where a function was intended.
- **Lazy evaluation and shortcut fusion.** The chaining wrapper (`_(value)`) builds
  a deferred sequence; when `.value()` is called, lodash fuses `map`/`filter`/`take`
  chains into a single pass and can short-circuit (e.g. stop after `take(3)`)[^4].
  This only applies to the explicit/implicit chaining API, not to standalone calls.
- **Guarded, normalized behavior.** Methods coerce and guard arguments (e.g.
  treating `null`/`undefined` as empty collections) for consistent results across
  engines. This normalization is exactly what native methods do *not* do, and is
  part of why lodash is bigger than a naive reimplementation.

The `lodash/fp` module rearranges every method to be data-last, auto-curried, and
immutable, so functions compose cleanly with `_.flow`. It is a different mental
model from the main API and does not mix well with it.

## Production Notes

**Bundle size is the headline footgun.** `import _ from "lodash"` pulls the entire
library into your bundle even if you use one function. Mitigations, roughly in order
of preference: use `lodash-es` with a tree-shaking bundler; import per-method
(`lodash/debounce`); or add `babel-plugin-lodash` + `lodash-webpack-plugin` to
strip unused features. The standalone `lodash.debounce`-style per-method packages
still exist on npm but are effectively frozen and no longer recommended as the
primary path.

**Security history is real and worth knowing.** Lodash has shipped several
high-profile prototype-pollution advisories in deep-path/merge methods —
CVE-2018-3721 and CVE-2018-16487 (`_.merge`/`defaultsDeep`), CVE-2019-10744
(`defaultsDeep`), and CVE-2020-8203 (`_.set`/`zipObjectDeep`)[^5] — plus
CVE-2021-23337, a command-injection issue in `_.template`[^6]. All are fixed in
current 4.17.x, but they mean: keep lodash patched, never feed untrusted keys into
`_.set`/`_.merge`/`_.zipObjectDeep`, and treat `_.template` as unsafe for
attacker-controlled input.

**Version confusion.** The long-standing npm-stable release is 4.17.21 (2021); the
`main` branch README now reads a higher `v4.x` number reflecting the governance
reboot. Pin to what npm actually resolves and check the changelog before assuming a
newer semver is published.

**`isEqual`/`cloneDeep` are not free.** Deep-equality and deep-clone over large or
cyclic structures are common performance surprises. Where the runtime supports it,
`structuredClone` covers many `cloneDeep` cases natively (with different semantics
for functions/DOM nodes). Benchmark before assuming lodash is the fastest option —
newer libraries specifically target these paths.

## When to Use / When Not

**Use when:**
- You need `debounce`/`throttle`, deep `merge`/`get`/`set` with dynamic string
  paths, or robust `cloneDeep`/`isEqual`, and want battle-tested edge-case handling.
- You're on an older toolchain/target where native array/object methods are missing
  or inconsistent.
- A codebase already standardizes on lodash and consistency beats shaving kilobytes.

**Avoid when:**
- You'd reach for it only for `map`/`filter`/`find`/`includes`/`flat` — modern JS
  and TypeScript cover these; see you-dont-need/You-Dont-Need-Lodash-Underscore.
- Bundle size is a hard constraint and you can't guarantee tree-shaking.
- You want first-class TypeScript inference and immutability — the FP-first
  alternatives are designed for it and lodash's types are external (`@types/lodash`).

## Alternatives

- es-toolkit/es-toolkit — modern, smaller, faster lodash-compatible utility set with built-in types; use when you want lodash ergonomics with less weight.
- remeda/remeda — TypeScript-first, tree-shakeable, data-last; use when type inference and FP composition matter most.
- ramda/ramda — auto-curried, immutable, purely functional; use when you're building point-free pipelines.
- toss/es-toolkit and radashi-org/radashi — zero-dependency modern helpers; use when you want a lean, actively-growing toolkit.
- you-dont-need/You-Dont-Need-Lodash-Underscore — not a library but a reference; use when you want to delete lodash in favor of native code.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2012-04 | Initial release as an Underscore.js fork[^1]. |
| 1.0.0 | 2013-01 | First stable line; Underscore-compatible. |
| 2.0.0 | 2013-09 | AMD support, semver-major cleanups. |
| 3.0.0 | 2015-01 | Lazy evaluation and shortcut fusion introduced[^4]. |
| 4.0.0 | 2016-01 | Full modular rewrite; per-method packages, `lodash/fp`, `lodash-es`. |
| 4.17.15 | 2019-07 | Includes CVE-2019-10744 prototype-pollution fix[^5]. |
| 4.17.21 | 2021-02 | Long-standing npm-stable; CVE-2021-23337 `_.template` fix[^6]. |
| — | 2025 | OpenJS + Sovereign Tech Agency funding; Feature-Complete stage, TSC reboot[^2]. |

## References

[^1]: lodash — origin as an Underscore.js fork by John-David Dalton. https://github.com/lodash/lodash/wiki/Changelog
[^2]: OpenJS Foundation blog, "Sovereign Tech Agency supports Lodash." https://openjsf.org/blog/sta-supports-lodash
[^3]: lodash README — builds, module formats, `lodash-cli`. https://github.com/lodash/lodash/blob/main/README.md
[^4]: Filip Zawada, "Faster JavaScript with lodash lazy evaluation" (lazy sequences / shortcut fusion, lodash 3.0). https://github.com/lodash/lodash/wiki/FP-Guide
[^5]: Snyk — lodash prototype pollution advisories (CVE-2018-3721, CVE-2018-16487, CVE-2019-10744, CVE-2020-8203). https://security.snyk.io/package/npm/lodash
[^6]: CVE-2021-23337 — command injection in `lodash` `_.template`. https://nvd.nist.gov/vuln/detail/CVE-2021-23337

## Tags

javascript, utility-library, functional-programming, npm, tree-shaking, lodash, underscore-fork, esm, prototype-pollution, browser
