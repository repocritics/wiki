# unjs/defu

> Recursively assign default properties to an object — the config-layering
> primitive underneath Nuxt, Nitro, and much of the unjs ecosystem.

[GitHub repo](https://github.com/unjs/defu) ·
[npm package](https://www.npmjs.com/package/defu) ·
[License: MIT](https://github.com/unjs/defu/blob/main/LICENSE)

## Overview

defu does one thing: given a target object and one or more defaults objects, it
returns a new object where missing (nullish) properties are filled in from the
defaults, recursing into nested plain objects[^1]. It is not a general-purpose
deep merge — it is *defaults application*, with a fixed, opinionated set of
rules: leftmost arguments win, `null`/`undefined` in the target are treated as
"absent" and get replaced by defaults, and arrays are concatenated rather than
replaced. Written in TypeScript, dependency-free, and small enough that bundle
cost is a non-issue.

The project is part of unjs, the framework-agnostic tooling collective that
grew out of Nuxt[^2]. That origin explains its shape: layered configuration
(user config over preset over framework defaults) is exactly the
`defu(user, layer, defaults)` call, and defu is what unjs's config loader c12
uses to merge configuration layers[^3]. The star count (~1.4k) wildly
understates its footprint — as a transitive dependency of Nuxt, Nitro, c12,
and dozens of other unjs packages, its npm download volume is orders of
magnitude beyond what its GitHub numbers suggest.

The defining tradeoff: its merge semantics are fixed and surprising to anyone
expecting lodash or `Object.assign` behavior. Precedence is inverted (leftmost
wins, not rightmost), explicit `null` cannot override a default, and array
concatenation regularly ambushes users who expect replacement. The escape
hatch is `createDefu`, which swaps the merge rule per key.

## Getting Started

```bash
npm install defu
```

```js
import { defu } from "defu";

// leftmost has priority; nested objects merge; missing keys filled in
defu({ a: { b: 2 } }, { a: { b: 1, c: 3 } });
// => { a: { b: 2, c: 3 } }

// arrays CONCAT (target items first) — they do not replace
defu({ array: ["b", "c"] }, { array: ["a"] });
// => { array: ["b", "c", "a"] }

// custom merge rule: replace arrays instead of concatenating
import { createDefu } from "defu";
const replaceArrays = createDefu((obj, key, value) => {
  if (Array.isArray(obj[key]) && Array.isArray(value)) {
    obj[key] = value;
    return true; // handled — skip default behavior
  }
});
```

## Architecture / How It Works

The entire runtime is one short recursive function[^4]. Each level shallow-
copies the defaults object (`Object.assign({}, defaults)`), then walks the
target's keys: nullish values are skipped (so the default survives), two
arrays are concatenated with the target's items first, two *plain* objects
recurse, and anything else is assigned from the target wholesale. `__proto__`
and `constructor` keys are skipped outright to block prototype-pollution
attacks through merged input[^1].

The plain-object gate matters: `Date`, `RegExp`, `Map`, class instances, and
functions are never merged — they replace the default by reference. This is
usually what you want for config values, but it means defu is not a deep
cloner: any branch that exists on only one side is carried into the result *by
reference*, so mutating the returned object can mutate your defaults object.

Three variants are built on the same `createDefu` hook: `defu` (standard),
`defuFn` (a function in the target is called with the default value and its
return value wins — useful for transforming defaults), and `defuArrayFn`
(same, but only when the default is an array). Since v6 these ship as named
exports from a dual CJS/ESM build[^5]. Type-level merge inference (exposed as
the `Defu` type utility) mirrors the runtime rules, so the inferred result
type of a merge chain matches what you actually get[^1].

## Production Notes

- **Array concat is the #1 footgun.** A user-supplied `plugins: ["mine"]`
  over a default `plugins: ["builtin"]` yields both, not a replacement — a
  recurring confusion in Nuxt module/config land. There is no option flag;
  replace semantics require `createDefu` as shown above.
- **You cannot express "explicitly off".** `defu({ timeout: null }, { timeout:
  5000 })` gives `5000`; nullish always means "use the default"[^1]. If your
  config schema needs `null` as a meaningful value, encode it differently
  (e.g. `false`, a sentinel string) or use a different merger.
- **Precedence reads backwards.** Leftmost wins — the opposite of
  `Object.assign` and most merge libraries where later arguments override.
  Argument-order bugs pass type checking and produce quietly wrong configs.
- **Shared references.** The result is not a deep clone; freeze it or clone it
  before mutation if your defaults object is long-lived (module-level default
  configs are the classic case).
- **Type-level cost.** The `Defu` utility recursion over large, deeply nested
  config types adds tsc work and can widen types in ways that need manual
  annotation; for hot paths in type-heavy codebases, annotate the result.
- **Maintenance profile.** Activity is steady but low-volume, which fits a
  ~100-line finished utility: v6 has been the API since March 2022, patch
  releases landed through April 2026, and the repo saw pushes as recently as
  July 2026[^6]. 26 open issues is mostly semantics discussion, not breakage.

## When to Use / When Not

**Use when:**
- Layering configuration: user options over preset over framework defaults,
  especially in Nuxt/unjs-adjacent code where defu is already in the tree.
- You want defaults semantics (fill in missing values) rather than merge
  semantics (later values override), with nested objects handled.
- You need a tiny, dependency-free, prototype-pollution-safe utility with
  type-level merge inference.

**Avoid when:**
- You need arrays to replace rather than concatenate and don't want to carry a
  `createDefu` wrapper.
- `null` must be a meaningful override value in your schema.
- You need deep cloning, circular-reference handling, or merging of non-plain
  objects (Dates, Maps, class instances) — defu assigns these by reference.
- You want conventional rightmost-wins merge; a merge library is a better fit
  than inverting your argument order mentally.

## Alternatives

- lodash/lodash — `_.defaultsDeep` when lodash is already a dependency; note
  it mutates its first argument and does not concatenate arrays.
- TehShrike/deepmerge — use instead when you want rightmost-wins merge
  semantics with a pluggable `arrayMerge` option.
- fastify/deepmerge — performance-focused deep merge with configurable array
  handling, for hot paths merging many objects.
- RebeccaStevens/deepmerge-ts — use when precise inferred result types across
  complex merges are the priority and you want merge, not defaults, behavior.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.0.1 | 2019-02 | Initial release. |
| v3.2.0 | 2020-11-09 | Type inference for merge results; namespace passed to custom mergers[^6]. |
| v4.0.0 | 2021-05-12 | Breaking: module exports rework[^6]. |
| v5.0.0 | 2021-05-12 | Breaking: nullish values in the source are skipped — the current "null means use the default" semantics — diverging from defaults-deep behavior. Released same day as v4[^6]. |
| v6.0.0 | 2022-03-21 | Breaking: named exports replace the default export; array concat order flipped so target items come first[^5]. |
| v6.1.4 | 2024-01-05 | Maintenance release. |
| v6.1.7 | 2026-04-07 | Latest release; fixes on the stable v6 API[^6]. |

## References

[^1]: defu README — usage, remarks on nullish handling, array concat, and `__proto__`/`constructor` skipping. https://github.com/unjs/defu#readme
[^2]: unjs — unified JavaScript tools collective. https://unjs.io
[^3]: unjs/c12 — configuration loader that merges layers with defu. https://github.com/unjs/c12
[^4]: defu source, `src/defu.ts`. https://github.com/unjs/defu/blob/main/src/defu.ts
[^5]: defu v6.0.0 release. https://github.com/unjs/defu/releases/tag/v6.0.0
[^6]: defu releases. https://github.com/unjs/defu/releases

## Tags

typescript, javascript, defaults, deep-merge, object-merging, configuration, utility, unjs, nuxt-ecosystem, zero-dependency
