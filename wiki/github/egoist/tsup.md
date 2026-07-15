# egoist/tsup

> Zero-config bundler for TypeScript libraries, wrapping esbuild — now officially unmaintained, with its author pointing users to tsdown.

[GitHub repo](https://github.com/egoist/tsup) ·
[Official website](https://tsup.egoist.dev) ·
[License: MIT](https://github.com/egoist/tsup/blob/main/LICENSE)

## Overview

tsup is a command-line bundler aimed specifically at TypeScript *libraries* (packages you publish to npm), not applications. Its pitch was that shipping a dual ESM/CJS package with type declarations normally requires assembling esbuild or Rollup plus a declaration-emit step plus config, and tsup collapses all of that into one command with sane defaults. Point it at an entry file, pass `--format esm,cjs --dts`, and it produces bundled JavaScript in both module systems plus a bundled `.d.ts`. It was created by EGOIST (Kevin Titor) in 2020 and became one of the default answers to "how do I bundle a TS library" for several years.

The defining tension is the split between *bundling* and *type emitting*. tsup delegates all JavaScript work to esbuild[^1], which is extremely fast but deliberately does no type checking and cannot emit `.d.ts` files. So the declaration output is produced by a separate, much slower pipeline (historically the TypeScript compiler API plus rollup-plugin-dts for bundling). In practice this means the `--dts` step dominates build time and behaves differently from the JS step — the single most common source of confusion and slow builds in tsup projects.

As of its current README, **the project is no longer actively maintained**[^2]. The author recommends migrating to tsdown, a Rolldown-based successor, and links a migration guide. The repo still receives occasional commits and has a large open-issue backlog (400+), but new users should read the deprecation notice before adopting it.

## Getting Started

```bash
npm i tsup -D          # or: pnpm add tsup -D / yarn add tsup --dev
```

```bash
# Bundle a library to dist/ in both ESM and CJS, with type declarations
tsup src/index.ts --format esm,cjs --dts --clean
```

```ts
// tsup.config.ts — checked-in config instead of long CLI flags
import { defineConfig } from "tsup";

export default defineConfig({
  entry: ["src/index.ts"],
  format: ["esm", "cjs"],
  dts: true,            // emit bundled .d.ts (slow path — see below)
  sourcemap: true,
  clean: true,
  treeshake: true,
  external: ["react"],  // keep peer deps out of the bundle
});
```

The matching `package.json` typically wires `exports` so consumers resolve the right file per module system:

```jsonc
{
  "main": "./dist/index.js",       // CJS
  "module": "./dist/index.mjs",    // ESM
  "types": "./dist/index.d.ts",
  "exports": {
    ".": { "import": "./dist/index.mjs", "require": "./dist/index.js" }
  }
}
```

## Architecture / How It Works

tsup is best understood as an opinionated orchestration layer over esbuild, not a bundler in its own right. The pieces:

- **JS pipeline (esbuild).** All transpilation, bundling, tree-shaking, minification, target lowering, and format conversion (ESM/CJS/IIFE) run through esbuild. This is why the JS half of a build is near-instant. esbuild's own limitations are inherited: no type checking, limited support for some newer TS features until esbuild catches up, and esbuild's decorator/emit semantics rather than `tsc`'s.
- **Declaration pipeline (separate).** Because esbuild cannot produce `.d.ts`, `--dts` runs a distinct process using the TypeScript compiler plus rollup-plugin-dts to bundle the declarations into one file. This path is single-threaded, type-aware, and slow relative to esbuild — often 5–20× the JS build time on real libraries. It can also fail on type errors that the fast JS build silently ignored.
- **Code splitting.** Enabled by default for ESM output when multiple entries share code; produces shared chunks. CJS splitting is more constrained by the format.
- **Config resolution.** `tsup.config.ts` is loaded and can export a function or array (multiple build configs). CLI flags override config. The config file itself is executed, so it can be dynamic.
- **esbuild plugins & hooks.** tsup exposes `esbuildPlugins`, `esbuildOptions`, and lifecycle hooks like `onSuccess` (a command or callback run after each successful build, commonly used to restart a dev process in watch mode).

The coupling story: tsup's ceiling is esbuild's ceiling. Anything esbuild can't express — complex output-format control, custom module graph transforms, granular chunking strategies — tsup can't either, short of dropping to raw esbuild options. This is intentional (the value is the defaults), but it is why teams with unusual output requirements eventually migrate to Rollup or, now, Rolldown/tsdown.

## Production Notes

- **`--dts` is the bottleneck and the footgun.** It is slow, memory-hungry on large type graphs, and can hang or OOM on projects with heavy generics or many re-exports. Common mitigations: emit declarations separately with `tsc --emitDeclarationOnly` and skip tsup's `--dts`, or use `experimentalDts` where available. Budget CI time around the dts step, not the JS step.
- **Bundled declarations differ from `tsc` output.** rollup-plugin-dts flattens types into one file; it occasionally mishandles complex conditional/mapped types or `namespace` merging, producing `.d.ts` that compiles differently than your source. Verify published types with a tool like `@arethetypeswrong/cli` before release.
- **ESM/CJS interop remains manual.** tsup emits both formats but does not fix fundamental dual-package hazards (a package loaded as both ESM and CJS getting two copies of module state). The `exports` map and `external` config are on you to get right; `.mjs`/`.cjs` extension choices matter for Node resolution.
- **esbuild version pinning.** tsup pins an esbuild version; upgrading tsup can silently change esbuild behavior (target lowering, tree-shaking edge cases). Lockfile-review tsup bumps as you would a compiler bump.
- **Watch mode + `onSuccess`.** `onSuccess` running a long-lived process needs care — older versions did not always kill the previous process cleanly on rebuild. Verify no orphaned processes in dev.
- **Maintenance risk is now the dominant caveat.** With the project declared unmaintained[^2], bugs against newer TypeScript/Node/esbuild versions may not be fixed. Treat tsup as frozen: fine for stable existing libraries, questionable for new projects that will need years of forward compatibility.

## When to Use / When Not

**Use when:**
- You maintain an existing library already on tsup and it works — there is no urgency to churn a stable build.
- You want the simplest possible dual ESM/CJS + types output for a small-to-medium TS library and accept a frozen tool.
- Build speed on the JS side matters and your type surface is modest.

**Avoid when:**
- You are starting a new library in 2026 — the author's own recommendation is tsdown[^2].
- You need fixes/support: the maintenance status means no reliable upstream.
- Your `.d.ts` output is large or type-heavy — the dts pipeline is the weak point.
- You need fine-grained control over chunking or output format — you'll fight esbuild's model.

## Alternatives

- rolldown/tsdown — the author-endorsed successor, built on Rolldown (Rust); use this for any new TS library today.
- unjs/unbuild — Rollup-based zero-config library bundler with a stub/dev mode and passive `.d.ts`; use when you want the UnJS ecosystem and Rollup's plugin depth.
- evanw/esbuild — the engine tsup wraps; use directly when you want full control and don't need the library-publishing conveniences.
- rollup/rollup — use when you need precise output/chunking control and a mature plugin ecosystem, and can tolerate more config.
- privatenumber/pkgroll — Rollup-based, reads your `package.json` `exports` to infer outputs; use when you want config driven entirely by `package.json`.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-03-19 | Repository created by EGOIST; esbuild-powered zero-config library bundler.[^1] |
| — | 2020–2025 | Became a default choice for TS library bundling; added dual-format output, `--dts`, code splitting, config file, esbuild plugin passthrough. |
| unmaintained | 2026 | README declares the project no longer actively maintained; recommends tsdown + migration guide.[^2] Last push 2026-06-14.[^3] |

## References

[^1]: tsup README — "Bundle your TypeScript library with no config, powered by esbuild." https://github.com/egoist/tsup
[^2]: tsup README deprecation notice — "This project is not actively maintained anymore. Please consider using tsdown instead," with link to https://tsdown.dev/guide/migrate-from-tsup . https://github.com/egoist/tsup
[^3]: GitHub API metadata for egoist/tsup — 11,281 stars, 272 forks, MIT license, created 2020-03-19, last push 2026-06-14 (retrieved 2026-07-15). https://api.github.com/repos/egoist/tsup

## Tags

typescript, javascript, bundler, esbuild, library-tooling, build-tool, dts, esm, cjs, npm-publishing, unmaintained, cli
