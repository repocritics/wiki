# lukeed/clsx

> A ~239-byte utility for building `className` strings from conditional strings, objects, and arrays.

[GitHub repo](https://github.com/lukeed/clsx) ·
[License: MIT](https://github.com/lukeed/clsx/blob/master/license)

## Overview

clsx is a single-purpose function: given any mix of strings, numbers, arrays,
and objects, it returns a space-joined `className` string with falsey values
discarded. It exists because writing `` `${a ? 'active' : ''} ${b && 'open'}` ``
by hand is error-prone (stray spaces, `undefined` leaking into the DOM), and
the incumbent solution — JedWatson's `classnames` — was slightly larger and
slower. clsx positions itself explicitly as a "faster and smaller drop-in
replacement" for that module[^1], with a compatible enough API that most
codebases can swap the import and move on.

The project is by Luke Edwards, whose body of work is a catalog of
sub-kilobyte utilities (obj-str, dequal, uvu, polka). clsx is the most widely
depended-upon of them, largely because Material UI (MUI) adopted it in place of
`classnames`, pulling it into a very large slice of the React ecosystem[^2]. It
has no runtime dependencies and ships in three formats — ESM, CommonJS, and
UMD — plus a stripped `clsx/lite` build.

The defining tension of a library this small is not features but discipline:
the value proposition is the byte count and the benchmark, so every addition is
weighed against them. That constraint is why `clsx/lite` exists as a separate
entry point rather than a flag, and why the core has stayed effectively frozen
for years. Nearly 10k stars on a function you could write in fifteen lines is a
statement about trust and defaults, not about surface area.

## Getting Started

```
$ npm install --save clsx
```

```js
import { clsx } from 'clsx';
// default import also works: import clsx from 'clsx';

clsx('foo', true && 'bar', 'baz');            //=> 'foo bar baz'
clsx({ foo: true, bar: false });              //=> 'foo'
clsx(['foo', 0, false, 'bar']);               //=> 'foo bar'
clsx('a', ['b', { c: true, d: null }], 'e');  //=> 'a b c e'

// All falsey inputs and standalone booleans are dropped:
clsx(true, false, '', null, undefined, 0, NaN); //=> ''
```

## Architecture / How It Works

The entire library is one exported function plus a small internal helper. The
public function iterates over `arguments` (variadic), and for each argument
dispatches on type: strings and numbers are appended as-is (numbers because
`0` is falsey and already filtered, so any surviving number is truthy); arrays
are recursed into via the same logic; objects have their keys appended when the
corresponding value is truthy. Falsey values and standalone booleans contribute
nothing. The result is accumulated into a single string with single-space
separators, avoiding leading/trailing whitespace by construction.

There is no parsing, no regex, no deduplication, and no ordering guarantee
beyond argument/insertion order. clsx does **not** merge or resolve conflicting
classes — `clsx('p-2', 'p-4')` returns `'p-2 p-4'`, both intact. That job
belongs to a separate layer (see Alternatives). Keeping clsx purely additive is
what lets it stay tiny and fast.

`clsx/lite` is a second, smaller entry point (~140 B gzip) that accepts
**only** string arguments and silently ignores objects and arrays[^1]. It
exists for the extremely common Tailwind pattern
`clsx('base', cond && 'variant', props.className)`, where the object/array
branches are dead weight. Choosing `lite` is an explicit opt-in to a narrower
contract, not a config toggle — consistent with the library's "bring only what
you need" framing.

Since v2.0.0 the package exposes a modern `exports` map so bundlers pick the
ESM build automatically while Node CommonJS `require` still resolves[^3]. The
UMD build (`dist/clsx.min.js`) remains for script-tag / no-bundler usage.

## Production Notes

- **It does not deduplicate or merge.** Passing the same class twice yields it
  twice; passing conflicting Tailwind utilities yields both. If you rely on
  "last class wins" for Tailwind, clsx alone will not give it to you — you need
  `dcastil/tailwind-merge` (often composed as `twMerge(clsx(...))`, which is
  effectively what the popular `cn()` helper does).
- **`clsx/lite` failure mode is silent.** If a developer passes an object to
  the lite build expecting object semantics, it is dropped with no warning and
  the class simply never appears. This surfaces as a missing style, not an
  error. Standardize on one entry point per project to avoid the trap.
- **Type surface.** The bundled TypeScript types accept the recursive
  `ClassValue` union (string, number, boolean, null, undefined, array, and a
  record). Editors will not stop you from passing a nested structure to the
  lite build, since the type is shared — the runtime silently ignores it.
- **Old-browser support is version-gated.** v1.x supports IE9+ via
  `Array.isArray`; IE8 and older require pinning to `clsx@1.0.x`, which carries
  a known issue[^4]. Modern targets are unaffected.
- **Effectively frozen.** The last release is v2.1.1 (April 2024) and the
  default branch has not been pushed since mid-2024. For a utility of this
  scope that reflects completeness rather than abandonment — there is little
  left to change — but expect issues to sit and do not plan around new features.

## When to Use / When Not

**Use when:**
- You conditionally assemble `className` strings in React/JSX (or anywhere) and
  want the falsey-filtering handled correctly and cheaply.
- You are migrating off `classnames` and want a smaller, faster, API-compatible
  replacement.
- You use Tailwind with the string-builder pattern — reach for `clsx/lite`.

**Avoid when:**
- You need conflicting utility classes resolved (Tailwind "last wins"): pair
  with `tailwind-merge` or use a `cn()` wrapper; clsx alone will not do it.
- You need only object-shaped input and want the absolute minimum: `obj-str`
  is smaller.
- You are on a framework with first-class conditional-class syntax (Svelte
  `class:`, Vue `:class` bindings) where a helper adds nothing.

## Alternatives

- JedWatson/classnames — the original and still widely used; use it when you
  want the incumbent with the largest install base and don't care about the
  byte/perf delta.
- lukeed/obj-str — same author, ~96 B, object-only; use it when every input is
  an object map and you want the smallest possible footprint.
- dcastil/tailwind-merge — solves a different problem (conflict resolution),
  not a replacement; compose it with clsx when you need Tailwind "last wins".
- lukeed/clsx `/lite` — the built-in string-only subset; use it when your
  usage is purely `clsx('a', cond && 'b')`.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2018-12 / 2019-01 | Initial release; classnames-compatible core[^1]. |
| 1.1.0 | 2020-02-03 | Iterative improvements to the core. |
| 1.2.0 | 2022-07-02 | Maintenance release. |
| 2.0.0 | 2023-07-16 | `exports` map, modern dual ESM/CJS packaging[^3]. |
| 2.0.1 | 2023-12-29 | Packaging fixes. |
| 2.1.0 | 2023-12-29 | `clsx/lite` string-only entry point added. |
| 2.1.1 | 2024-04-23 | Latest release; type/packaging touch-ups. |

## References

[^1]: clsx README — API, modes, and "faster & smaller drop-in replacement for `classnames`" framing. https://github.com/lukeed/clsx#readme
[^2]: MUI (Material UI) uses clsx internally for className composition. https://mui.com/
[^3]: clsx `package.json` `exports` map, introduced in the v2.0.0 line. https://github.com/lukeed/clsx/blob/master/package.json
[^4]: clsx issue #17 — IE8 support caveat referenced by the README's Support section. https://github.com/lukeed/clsx/issues/17

## Tags

javascript, react, classnames, css, utility, frontend, tailwind, zero-dependency, tiny, dom
