# unplugin/unplugin-auto-import

> Build-time plugin that injects imports for APIs you use without writing the import statements — for Vite, webpack, Rollup, Rspack, Rolldown, and esbuild.

[GitHub repo](https://github.com/unplugin/unplugin-auto-import) ·
[License: MIT](https://github.com/unplugin/unplugin-auto-import/blob/main/LICENSE)

## Overview

`unplugin-auto-import` is a bundler plugin that scans your source files, finds usages of APIs you have registered (e.g. `ref`, `computed`, `useState`), and injects the corresponding `import` statements at transform time. The goal is to remove import boilerplate for a fixed vocabulary of frequently-used functions from Vue, React, vue-router, `@vueuse/core`, and arbitrary user-defined modules. It was created by Anthony Fu (`antfu`) in 2021 and later moved into the `unplugin` GitHub organization; the npm package remains `unplugin-auto-import`[^1].

It is built on two layers of `antfu`/`unjs` infrastructure. `unplugin` provides one plugin definition that runs unchanged across Vite, webpack, Rollup, Rspack, Rolldown, and esbuild[^2]. Since v0.8.0 the actual import-resolution and code transformation is delegated to `unimport`, a lower-level engine also used by Nuxt's built-in auto-import[^3]. `unplugin-auto-import` is best understood as the user-facing config wrapper around `unimport` — it adds resolvers, `.d.ts` generation, and ESLint/Biome globals files.

The defining tension is implicitness. Removing imports makes files shorter but breaks the assumption that every identifier is traceable to a declaration in the file. Go-to-definition, grep-ability, and "where did this come from" all now depend on generated type files and on the reader knowing the auto-import vocabulary. The plugin is most valuable in codebases (Vue SFCs especially) where the same dozen APIs appear in every file, and least valuable in code read by people unfamiliar with the convention.

## Getting Started

```bash
npm i -D unplugin-auto-import
```

```ts
// vite.config.ts
import AutoImport from 'unplugin-auto-import/vite'
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [
    AutoImport({
      imports: ['vue', 'vue-router'],
      dts: './auto-imports.d.ts', // generated type declarations
      eslintrc: { enabled: true }, // generate globals for ESLint no-undef
    }),
  ],
})
```

After this, `ref`, `computed`, `watch`, `useRoute`, etc. can be used without importing them:

```ts
// src/counter.ts — no import line needed
const count = ref(0)
const doubled = computed(() => count.value * 2)
```

The entry point differs per bundler: `unplugin-auto-import/vite`, `/webpack`, `/rollup`, `/rspack`, `/rolldown`, `/esbuild`, and `/astro`. Nuxt users do not need this plugin — auto-import is built in.

## Architecture / How It Works

The plugin registers a set of **import bindings** — a map from identifier name to `(module, exportName)`. Bindings come from presets (`'vue'` expands to Vue's public API), from inline `imports` entries, from `resolvers` (function-based, compatible with `unplugin-vue-components`), and from scanning `dirs` (every module under `./composables`, etc.).

At transform time, for each file matching `include`, `unimport` parses the module, collects the free identifiers, matches them against the registered bindings, and prepends the needed imports. Only identifiers actually used are injected, so output is tree-shakeable — nothing is added to files that do not reference an auto-imported name. Vue `<template>` usage requires `vueTemplate: true` because template identifiers are not visible to the default source scan.

TypeScript support is a **build side effect**, not a runtime feature. When `dts` is enabled the plugin writes an `auto-imports.d.ts` file declaring every binding as a global. The type checker and editor read that file; it must exist and be current for types to resolve. It is regenerated on dev server start and on build. `dtsMode: 'append'` (the default) preserves manual additions; `'overwrite'` replaces the file wholesale.

ESLint's `no-undef` rule flags auto-imported names as undefined because, lexically, they are. The `eslintrc` option generates a `.eslintrc-auto-import.json` globals file to suppress this; `biomelintrc` does the same for Biome. The README recommends simply disabling `no-undef` under TypeScript, since `tsc` already covers undefined-variable checking.

Because the whole system is a wrapper, most non-trivial behavior — resolution order, caching, package preset detection — lives in `unimport` and is documented there, not here. New features increasingly land in `unimport` rather than this repo[^3].

## Production Notes

**The generated `.d.ts` is a race condition in CI.** `auto-imports.d.ts` is produced by running the bundler (dev or build). A pipeline that runs `tsc --noEmit` or `vue-tsc` *before* a build step will fail on a clean checkout because the declaration file does not exist yet. Standard fixes: commit the generated file to source control, or run a build/`unimport` generation step before type-checking. Committing it works but produces noisy diffs whenever the auto-import vocabulary changes.

**Commit-or-ignore, pick one and be consistent.** Teams that gitignore `auto-imports.d.ts` and `.eslintrc-auto-import.json` must generate them in every environment (CI, fresh clone, editor-only sessions). Teams that commit them accept churn. There is no third option; a half-committed state causes phantom type errors that differ per machine.

**Name collisions are silent-ish.** If two registered modules export the same name, or an auto-imported name shadows a local variable, resolution follows a precedence order that is easy to get wrong. Use the `ignore` option to exclude specific names, and `injectAtEnd` / `dts` inspection to diagnose. Collisions typically surface as "wrong function imported" rather than a hard error.

**Readability and onboarding cost is real.** New contributors cannot tell auto-imported globals from language builtins without knowing the config. Go-to-definition works only through the generated `.d.ts`, so a stale or missing file breaks editor navigation. Grep for an import to find usage sites no longer works for auto-imported APIs. This is the recurring complaint in code review of projects that adopt it heavily.

**Vite `optimizeDeps`.** Auto-imported packages can miss Vite's dependency pre-bundling because they never appear in a static import graph the scanner sees early. The `viteOptimizeDeps: true` option (recommended in the README) adds them explicitly to avoid dev-server reload churn.

**Version coupling.** Because behavior is delegated to `unimport` and the plugin surface to `unplugin`, a bug or behavior change can originate in any of three packages. When debugging, check the installed versions of all three, not just `unplugin-auto-import`.

## When to Use / When Not

**Use when:**
- You have a Vue SFC codebase where the same composition APIs appear in nearly every file.
- You control the whole team and the auto-import convention is documented and understood.
- You want on-demand, tree-shakeable injection rather than runtime globals (the successor pattern to `vue-global-api`).
- You are already on Vite/Rollup/webpack and want one config that also serves other bundlers.

**Avoid when:**
- The codebase is read by outside contributors or maintained long-term by rotating teams — implicitness raises onboarding cost.
- You need reliable pre-build type-checking in CI without extra generation steps.
- You use only a handful of imports; the boilerplate saved does not justify the tooling and the `.d.ts` lifecycle.
- You value grep-ability and stable go-to-definition over terseness.

## Alternatives

- unplugin/unplugin-vue-components — same author and pattern, but auto-imports Vue *components* (and directives) instead of JS APIs; commonly paired with this plugin.
- unjs/unimport — the lower-level engine this plugin wraps; use it directly when you need auto-imports outside a bundler plugin context.
- nuxt/nuxt — has auto-imports built in; do not add this plugin to a Nuxt project.
- antfu/vue-global-api — the runtime-global predecessor; use only if you specifically want zero build-tool integration and accept global pollution.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2021-08 | Created by Anthony Fu as `antfu/unplugin-auto-import`, built on `unplugin`[^1]. |
| 0.8.x | 2022 | Transform engine switched to `unimport`; new features move upstream[^3]. |
| — | later | Repository moved to the `unplugin` GitHub organization; npm name unchanged. |
| current | 2026-01 | Supports Vite, webpack, Rollup, Rspack, Rolldown, esbuild, Astro[^2]. |

## References

[^1]: unplugin-auto-import README and license (© 2021–PRESENT Anthony Fu). https://github.com/unplugin/unplugin-auto-import
[^2]: unplugin — one plugin definition across Vite/webpack/Rollup/Rspack/Rolldown/esbuild. https://github.com/unjs/unplugin
[^3]: unimport — the auto-import engine; README FAQ: "From v0.8.0, unplugin-auto-import uses unimport underneath." https://github.com/unjs/unimport

## Tags

typescript, vite-plugin, webpack-plugin, rollup-plugin, bundler, build-tool, auto-import, developer-tooling, vue, unplugin
