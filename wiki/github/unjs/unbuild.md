# unjs/unbuild

> Zero-to-low-config build system for publishing JavaScript/TypeScript libraries — rollup + esbuild under the hood, config inferred from `package.json`.

[GitHub repo](https://github.com/unjs/unbuild) ·
[License: MIT](https://github.com/unjs/unbuild/blob/main/LICENSE)

## Overview

unbuild is the build tool of the UnJS ecosystem (the collection of framework-agnostic packages behind Nuxt: h3, ofetch, jiti, nitro, and others). It targets one job precisely: turning a library's `src/` into a publishable `dist/` with dual CommonJS/ESM output and type declarations, while requiring as little configuration as possible. Its defining trick is inference — entries, output formats, and declaration settings are derived from the `exports`, `main`, and `types` fields of `package.json`, so a conventional package often needs no config file at all[^1].

The repo was created in April 2021 and reached a stable 1.0 in November 2022[^2]. It is the standard build tool across UnJS packages and, via `@nuxt/module-builder`, for most of the Nuxt module ecosystem — which means its real install base is far larger than its ~2.7k stars suggest. As of mid-2026 it is actively maintained (last push July 2026), but the README itself points users experimenting with build speed toward **obuild**, a rolldown-based successor the team is incubating[^3]. That makes unbuild the stable, boring choice today, with a visible succession plan rather than an indefinite roadmap.

The core tension: unbuild is a bundler for *libraries*, not apps. It deliberately externalizes your runtime `dependencies`, checks for missing/unused ones, and fails CI on mismatches[^1]. If you want app-style "bundle everything," or bleeding-edge speed, its rollup foundation and opinions work against you.

## Getting Started

```sh
npx unbuild        # zero config: infers entries from package.json + src/
```

With explicit configuration, create `build.config.ts`:

```ts
import { defineBuildConfig } from "unbuild";

export default defineBuildConfig({
  entries: [
    "./src/index",                                  // rollup builder (default)
    { builder: "mkdist", input: "./src/runtime/" }, // file-to-file transpile
  ],
  declaration: true,   // emit .d.ts / .d.mts / .d.cts
  sourcemap: true,
});
```

Typical `package.json` wiring:

```json
{
  "type": "module",
  "exports": { ".": { "import": "./dist/index.mjs", "require": "./dist/index.cjs" } },
  "main": "./dist/index.cjs",
  "types": "./dist/index.d.ts",
  "files": ["dist"],
  "scripts": { "build": "unbuild", "prepack": "unbuild" }
}
```

## Architecture / How It Works

unbuild is an orchestration layer over other UnJS and rollup-ecosystem pieces rather than a bundler written from scratch:

- **Rollup builder (default)** — bundles each entry with rollup; TypeScript/modern syntax is transpiled by esbuild inside the rollup pipeline. Emits `.mjs` and (when configured) `.cjs` per entry.
- **Declaration generation** — `.d.ts` output via a rollup-based d.ts pipeline. The `declaration` option has real semantics: `"compatible"` emits `.d.mts` + `.d.cts` + `.d.ts`; `"node16"` emits only `.d.mts`/`.d.cts`; `undefined` auto-detects from the presence of a `types` field[^1].
- **mkdist builder** — file-to-file transpilation without bundling, for cases where source structure must survive into `dist/` (Vue components, Nuxt runtime directories)[^4].
- **Stub mode** — `unbuild --stub` writes tiny dist files that load your *source* through jiti at require/import time[^5]. This replaces a watch process entirely: link the package once, edit `src/`, and consumers see changes immediately. It is unbuild's most distinctive feature and its biggest footgun (see below).
- **Dependency hygiene** — imports are resolved against `package.json`: `dependencies` are externalized, anything imported but undeclared is flagged as a potential missing dependency, unused declared dependencies are also flagged, and CI fails on violations[^1]. The build summary prints output sizes and exports.
- **untyped integration** — generates types + markdown from config schemas, used mostly inside the UnJS/Nuxt world[^6].

The coupling story is honest: unbuild's behavior is the sum of rollup, esbuild, jiti, and mkdist. When you hit an edge case, the fix frequently lives in one of those upstream packages, and debugging requires knowing which layer you are in.

## Production Notes

- **Stub mode is dev-only and leaky.** Stubbed `dist/` files depend on jiti resolving your source at runtime. Anything that consumes the package outside a jiti-friendly Node process — a bundler with strict resolution, an edge runtime, `npm pack` — will break or silently ship stubs. Always run a real `unbuild` in `prepack` so publishing never depends on a developer remembering.
- **esbuild transpiles, it does not type-check.** Type errors do not fail the build. Run `tsc --noEmit` (or `vue-tsc`) as a separate CI step or you will publish broken types with a green build.
- **d.ts generation is the slow path.** For packages with large or deeply generic public types, declaration bundling dominates build time and memory, a known cost of rollup-based d.ts pipelines. Mitigation: `declaration: false` during iteration, full builds only in CI/prepack.
- **Dependency checks bite monorepos.** Imports satisfied by hoisting but not declared in the package's own `package.json` fail the build. This is the tool working as designed — it catches real publish bugs — but expect a cleanup pass when adopting it in an existing workspace.
- **Decorators and other tsconfig nuances need explicit wiring.** esbuild consumes only part of `tsconfig.json`; features like `experimentalDecorators` must be set via `rollup.esbuild.tsconfigRaw` in the build config[^1].
- **v3 landed after a year-long RC.** v3.0.0-rc.1 shipped December 2023; stable v3.0.0 arrived December 2024[^2]. Release cadence since has been steady but unhurried (v3.6.1, August 2025). Combined with the obuild note in the README, plan for unbuild to stay maintained-stable rather than fast-moving — and watch obuild before making new long-term tooling bets[^3].

## When to Use / When Not

**Use when:**
- You are publishing a TypeScript/JS library to npm and want dual ESM/CJS + declarations with near-zero config.
- You are writing a Nuxt module or UnJS-adjacent package — it is the ecosystem default and templates assume it.
- You want CI to catch missing/undeclared dependencies before your users do.
- You value the `--stub` workflow for iterating on a linked package without a watcher.

**Avoid when:**
- You are building an application — use Vite, webpack, or your framework's tooling; unbuild's externalize-dependencies model is wrong for apps.
- Raw build speed on a large codebase is the priority — esbuild-native tools (tsup) or rolldown-based ones (obuild, tsdown) are faster than the rollup pipeline.
- You need heavy custom bundler behavior — at that point, configuring rollup directly is clearer than tunneling options through `rollup.*` config keys.
- You need a tool with a guaranteed decade-long roadmap — the maintainers are openly incubating a successor.

## Alternatives

- egoist/tsup — use instead when you want esbuild-native speed and simple lib bundling without unbuild's package.json inference and dependency checks.
- unjs/obuild — the experimental rolldown-based successor from the same team; use when you accept beta software for faster builds[^3].
- rollup/rollup — use directly when your bundling needs outgrow unbuild's config surface.
- developit/microbundle — older zero-config lib bundler; use for tiny packages if you are already invested in it (less active development).
- vitejs/vite — library mode; use when the library ships alongside a Vite app and you want one toolchain.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2021-04 | Repo created inside UnJS; rapid pre-1.0 iteration[^2]. |
| 1.0.0 | 2022-11-16 | First stable release[^2]. |
| 2.0.0 | 2023-08-22 | Major-version bump[^2]. |
| 3.0.0-rc.1 | 2023-12-18 | Start of a year-long v3 RC cycle[^2]. |
| 3.0.0 | 2024-12-13 | v3 stable[^2]. |
| 3.5.0 | 2025-02-26 | Feature release[^2]. |
| 3.6.1 | 2025-08-15 | Latest release as of this writing[^2]. |

## References

[^1]: unbuild README — usage, configuration, declaration semantics, dependency checks. https://github.com/unjs/unbuild#readme
[^2]: unbuild GitHub releases and tags (v1.0.0 tag dated 2022-11-16). https://github.com/unjs/unbuild/releases
[^3]: obuild — rolldown-based next-gen successor, linked from the unbuild README. https://github.com/unjs/obuild
[^4]: mkdist — file-to-file transpiler used by the mkdist builder. https://github.com/unjs/mkdist
[^5]: jiti — runtime TS/ESM loader powering `unbuild --stub`. https://github.com/unjs/jiti
[^6]: untyped — schema-to-types generator integrated by unbuild. https://github.com/unjs/untyped

## Tags

typescript, javascript, build-tool, bundler, rollup, esbuild, npm-publishing, library-tooling, unjs, dual-package, dts-generation
