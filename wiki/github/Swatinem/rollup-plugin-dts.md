# Swatinem/rollup-plugin-dts

> A Rollup plugin that bundles many `.d.ts` files into one rolled-up declaration
> file — by tricking Rollup into tree-shaking TypeScript types as if they were
> JavaScript.

[GitHub repo](https://github.com/Swatinem/rollup-plugin-dts) ·
License: LGPL-3.0

## Overview

rollup-plugin-dts solves a gap TypeScript itself has left open since 2015:
`tsc --declaration` emits one `.d.ts` per source file, but library authors
shipping a single bundled `index.js` usually want a single bundled `index.d.ts`
to match. A TypeScript proposal to flatten declarations exists but has seen no
progress[^1], so the ecosystem filled the hole with third-party bundlers. This
plugin, started by Arpad Borsos (Swatinem) in November 2018, became the most
widely used of them — largely invisibly, because tools like `tsup` and
Vite-ecosystem build chains embed it as their declaration-bundling backend.

At ~870 stars it looks mid-sized, but its npm reach is far larger than the star
count suggests; it is transitive infrastructure for a substantial share of
published TypeScript packages. Two facts define its current posture: it is in
explicit **maintenance mode** (no new feature development; occasional releases
for TypeScript compatibility — TS6 compatibility landed in v6.4.0)[^2], and it
is licensed **LGPL-3.0** with the author on record that it will never be
relicensed to a permissive license[^3]. Neither fact has slowed adoption — a
build-time plugin never links into your shipped artifact, so the copyleft
rarely bites — but both are things a due-diligence pass will flag.

The push cadence (last push 2026-07-13, releases through v6.4.1 in March 2026)
shows maintenance mode means "slow but alive," not abandoned: 22 open issues,
community PRs still merged, TypeScript-compat releases still shipped.

## Getting Started

```bash
npm install --save-dev rollup-plugin-dts
```

```js
// rollup.config.js — a second config entry alongside your JS build
import { dts } from "rollup-plugin-dts";

export default [
  // ... your normal JS bundle config ...
  {
    input: "./dist/types/index.d.ts",   // tsc --declaration output
    output: [{ file: "dist/my-library.d.ts", format: "es" }],
    plugins: [dts()],
  },
];
```

Then point `package.json` at the bundle: `"types": "dist/my-library.d.ts"`.
Use the named import (`{ dts }`); the default import still works but triggers
`TypeError: dts is not a function` under some config-loading setups[^4].
External packages (including `@types/*`) are auto-marked external and excluded
from the bundle by default.

## Architecture / How It Works

The internals are an admitted abuse of Rollup, documented candidly by the
author[^5]. Rollup bundles by string manipulation (MagicString) plus AST-driven
dead-code elimination. The plugin exploits this: it parses declarations with
the TypeScript compiler API, then presents Rollup with a **virtual AST** of
bogus `FunctionDeclaration` nodes — one per exported `class`/`function`/
`interface`/`type` — each annotated with the `start`/`end` byte offsets of the
real declaration text. When Rollup tree-shakes an unreferenced "function," it
blindly deletes those bytes, removing the actual type declaration.

Cross-declaration type references are encoded as function default arguments
(`function bar(_0 = bar, _1 = foo) {}`) so Rollup's reference tracking and
identifier renaming (for name collisions across modules) operate on the right
byte ranges without inserting stray semicolons. Side effects are faked via
calls to unreferenced identifiers so Rollup cannot prove code removable.

Consequences of this design: the plugin does its **own import resolution
through the TypeScript compiler**, so combining it with `@rollup/plugin-
node-resolve` or other resolution plugins is explicitly unsupported and a
frequent source of confusing errors[^2]. Sourcemaps required real work —
Rollup's bundle maps are sparse (declaration-boundary granularity) while
TypeScript declaration maps are per-token, so v6.4.0 introduced a
"sparse-anchor hydration" scheme to restore Go-to-Definition fidelity[^5].

## Production Notes

- **Feed it `.d.ts`, not `.ts`.** The plugin works best on declarations already
  emitted by `tsc` from idiomatic code. Running it directly on `.ts`/`.tsx`
  (or `.js` with `allowJs`) works but is documented as not recommended[^2];
  the two-step `tsc --declaration` → dts-rollup pipeline is the reliable path.
- **Do not stack resolution plugins** (`node-resolve`, path-alias resolvers)
  in the dts config entry. Use the plugin's own `tsconfig` /
  `compilerOptions.paths` options for aliases instead.
- **Bundling external `@types` is a trap.** `respectExternal: true` pulls
  third-party types into your bundle; it "generally works" but is discouraged
  — you inherit their bugs, size, and license questions. The narrower
  `includeExternal: [...]` (v6.2.2) is the surgical option.
- **TS2742 on shared chunks.** With multiple entry points, a public type that
  originates in a private shared chunk can make downstream `tsc --declaration`
  fail with "type cannot be named." The plugin rewrites paths through a public
  re-export when one exists, but will not invent exports — you must re-export
  the type publicly yourself, or downstream consumers break[^2].
- **Version coupling.** Major versions track Rollup/TypeScript/Node minimums
  (v6 requires Rollup ≥3.25, TS ≥4.5; v6.1 added Rollup 4). When a new
  TypeScript release lands, expect a compatibility lag measured in weeks —
  maintenance mode's real cost.
- **Sourcemap output needs both switches**: `dts({ sourcemap: true })` and
  Rollup's `output.sourcemap: true`, or you silently get none.

## When to Use / When Not

**Use when:**
- You publish a TypeScript library and want one `.d.ts` matching your bundled
  JS entry, with private types tree-shaken out of the public surface.
- You already have a Rollup config — one extra config entry gets you there.
- You use tsup/similar and want to understand or patch the layer under it.

**Avoid when:**
- You don't bundle at all — shipping `tsc`'s per-file declarations is simpler
  and preserves perfect Go-to-Definition.
- You need API-surface governance (reports, release gates, doc extraction) —
  API Extractor does declaration rollup plus that, at higher config cost.
- You are not on Rollup: esbuild/rolldown/tsc-only pipelines have their own
  native or dedicated options.
- Your build depends on hand-written `.d.ts` with exotic module tricks — the
  virtual-AST approach targets idiomatic compiler-emitted declarations.

## Alternatives

- microsoft/rushstack — API Extractor: declaration rollup plus API reports and
  review workflow; use when you want governance, not just bundling.
- timocov/dts-bundle-generator — standalone CLI, no Rollup required; use for
  a tsc-based pipeline with no bundler in the loop.
- wessberg/rollup-plugin-ts — compiles TS and merges declarations in one
  Rollup pass; use when you want a single-pass build instead of two configs.
- egoist/tsup — esbuild-based bundler that wraps this plugin for its
  `--dts` flag; use when you want zero-config instead of raw Rollup.
- rolldown/tsdown — Rolldown-native successor path; use on new projects
  already betting on the Rolldown/oxc toolchain.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.0.0 | 2019-06-05 | First stable release[^6]. |
| v2.0.0 | 2020-11-22 | TS 4.1 target; pre-processing for `export default` and variable-declaration splitting[^7]. |
| v3.0.0 | 2021-03-06 | Native ESM package; Node 12, TS 4.2, Rollup 2.40[^7]. |
| v4.0.0 | 2021-08-29 | TS 4.4 / Rollup 2.56; nested namespace support[^7]. |
| v5.0.0 | 2022-10-15 | Rollup 3, Node 14[^7]. |
| v6.0.0 | 2023-08-17 | Rollup ≥3.25, TS ≥4.5, Node 18; `export { type T }` syntax[^7]. |
| v6.1.0 | 2023 | Rollup 4 compatibility[^7]. |
| v6.4.0 | 2026 | Per-token sourcemaps (Go-to-Definition), TS6 compatibility[^7]. |
| v6.4.1 | 2026-03-21 | Sourcemap regression fix; latest release[^7]. |

## References

[^1]: TypeScript issue #4433, "Provide a way to bundle declaration files" — open since 2015. https://github.com/Microsoft/TypeScript/issues/4433
[^2]: rollup-plugin-dts README — maintenance mode, expectations, TS2742 notes. https://github.com/Swatinem/rollup-plugin-dts#readme
[^3]: README, License section: "I have no intention to license this under any non-copyleft license." https://github.com/Swatinem/rollup-plugin-dts#license
[^4]: Issue #247 — `dts is not a function` with default import. https://github.com/Swatinem/rollup-plugin-dts/issues/247
[^5]: "How does it work" — virtual AST and sourcemap design doc. https://github.com/Swatinem/rollup-plugin-dts/blob/master/docs/how-it-works.md
[^6]: v1.0.0 tag, 2019-06-05. https://github.com/Swatinem/rollup-plugin-dts/tags
[^7]: CHANGELOG.md. https://github.com/Swatinem/rollup-plugin-dts/blob/master/CHANGELOG.md

## Tags

typescript, rollup, rollup-plugin, dts, declaration-bundling, build-tools, javascript-tooling, library-publishing, type-definitions, bundler
