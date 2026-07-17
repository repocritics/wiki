# unplugin/unplugin-vue-components

> Compile-time auto-import for Vue components: reference a component in a template and the plugin writes the import for you, on demand.

[GitHub repo](https://github.com/unplugin/unplugin-vue-components) ·
[npm](https://www.npmjs.com/package/unplugin-vue-components) ·
[License: MIT](https://github.com/unplugin/unplugin-vue-components/blob/main/LICENSE)

## Overview

unplugin-vue-components is a build-time plugin that removes the boilerplate of importing and registering Vue components. It scans your templates, matches PascalCase tags against components it can find on disk (or resolve from a UI library), and injects the corresponding `import` statements before the file is compiled. The result is tree-shakable — only components actually referenced end up in the bundle — with no runtime cost, because all the work happens during the build.

The project began life as `vite-plugin-components` by Anthony Fu in 2020, then was rewritten on top of [unplugin](https://github.com/unjs/unplugin) and renamed[^1]. That rewrite is the defining decision: instead of being a Vite plugin, it became a single codebase that emits Vite, Webpack, Rspack, Rollup, Rolldown, and esbuild plugins from one hook implementation. Most Vue starters that "just know" your components — notably antfu's Vitesse template — are wired through this plugin plus its sibling `unplugin-auto-import`.

The defining tension is magic versus explicitness. Auto-import makes template code shorter and is genuinely convenient at scale, but it moves component resolution into an opaque build step. A tag that resolves fine in one file may silently fail to resolve in another if the naming or directory convention drifts, and newcomers to a codebase get no `import` line to grep for. The plugin leans hard into convention; teams that value explicit dependency graphs sometimes deliberately turn it off.

## Getting Started

```bash
npm i unplugin-vue-components -D
```

```ts
// vite.config.ts
import Components from 'unplugin-vue-components/vite'
import { defineConfig } from 'vite'

export default defineConfig({
  plugins: [
    Components({
      dirs: ['src/components'],   // scanned directories (default)
      dts: true,                  // emit components.d.ts for TS support
    }),
  ],
})
```

```vue
<!-- No import, no components: {} registration needed -->
<template>
  <HelloWorld msg="Hello Vue 3" />
</template>
```

The same plugin is imported from `unplugin-vue-components/webpack`, `/rspack`, `/rollup`, `/rolldown`, or `/esbuild` for other bundlers. Note that CommonJS `require()` entry points were removed after v29.1.0[^2] — Webpack and Rspack configs must be ESM.

## Architecture / How It Works

The core is a source transform. During compilation the plugin receives each module's code, uses a template parser to find candidate component tags, and for each unresolved tag runs its resolution chain:

1. **Filesystem match** — components discovered by globbing `dirs`/`globs`. Folder structure can become a namespace prefix via `directoryAsNamespace` (e.g. `form/Input.vue` → `<FormInput>`).
2. **Resolvers** — functions that map a component name to an import path for a UI library. Built-in resolvers ship for Element Plus, Ant Design Vue, Vuetify, Naive UI, Vant, PrimeVue, and roughly two dozen others. A resolver is just `(name) => { name, from } | void`, so custom ones are trivial.
3. **Directives** — the same mechanism resolves `v-*` directives when a resolver supports them.

Matched tags get an `import` injected and the component wired into the module's registration. Because this is compile-time, an auto-imported component that lives under a lazily-loaded parent is code-split along with that parent for free.

The multi-bundler support comes entirely from unplugin: the plugin author writes one set of hooks (`transform`, `resolveId`, etc.) against unplugin's normalized interface, and unplugin adapts them to each bundler's plugin API. This is why the same package can target esbuild and Webpack without per-bundler forks — but it also means behavior is only as consistent as unplugin's abstraction, and the transform runs at slightly different pipeline stages depending on the host bundler.

For TypeScript, the plugin generates and continuously rewrites a `components.d.ts` file that augments Vue's `GlobalComponents` interface[^3], which is what gives editors type-checking and go-to-definition on tags that have no visible import.

## Production Notes

- **`components.d.ts` churn.** The generated declaration file changes whenever components are added, removed, or renamed. Committing it produces noisy diffs and occasional merge conflicts; not committing it means a clean checkout has broken types until the dev server regenerates it. Teams pick one convention and enforce it — there is no clean answer.
- **Only literal template tags resolve.** Resolution is static. `<component :is="dynamicName">` and components chosen at runtime are invisible to the scanner, so anything dynamic still needs a manual import or global registration. `excludeNames` exists precisely to suppress false matches on async/dynamic names.
- **Name collisions are silent-ish.** Two components with the same base name in different folders will collide unless you use `directoryAsNamespace` or `allowOverrides`. The failure mode is importing the wrong file, not an error.
- **Resolvers are effectively frozen.** The maintainers no longer accept new UI-library resolvers[^4]; several libraries (Vant, Varlet, TDesign) now ship their own first-party resolver packages, and Vuetify recommends its official Vite/Webpack plugins over this one. Check whether your UI library has a dedicated resolver before relying on the built-in.
- **Nuxt users usually don't need it.** Nuxt has built-in component auto-import; layering this plugin on top duplicates work and can conflict.
- **HMR edge cases.** Renaming a component file or changing directory-namespace options sometimes requires a dev-server restart before the declaration file and resolution catch up.

## When to Use / When Not

**Use when:**
- You have a large, flat-ish component library and want to cut import boilerplate across many files.
- You depend on a UI library with a supported resolver and want on-demand, tree-shaken imports without hand-writing them.
- You're building on Vite/Vitesse-style tooling and want the conventional Vue DX.

**Avoid when:**
- You're on Nuxt (use its built-in auto-import).
- Your team prefers explicit imports for auditability, or your codebase leans on dynamic `<component :is>`.
- Your UI library ships its own official plugin/resolver — prefer that.

## Alternatives

- unplugin/unplugin-auto-import — sibling plugin for auto-importing functions/APIs (Vue, VueUse, etc.) rather than components; commonly used alongside this one.
- nuxt/nuxt — has first-party component auto-import built in; no separate plugin needed.
- vuejs/core — plain explicit `import` + `components: {}` (or `<script setup>`); zero magic, fully greppable, the baseline this plugin replaces.
- unplugin/unplugin-icons — same author/ecosystem, auto-imports icon sets as components; pairs with this plugin's resolver mechanism.
- vuetifyjs/vuetify — ships `vite-plugin-vuetify` / `webpack-plugin-vuetify`, the recommended path for Vuetify instead of the built-in resolver.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2020-08-20 | Created as `vite-plugin-components` (Vite-only)[^1]. |
| ~0.14 | 2021 | Rewritten on unplugin, renamed to `unplugin-vue-components`, multi-bundler[^1]. |
| — | ongoing | Resolvers frozen to new additions; libraries move to first-party resolvers[^4]. |
| >29.1.0 | 2025 | CommonJS entry points removed; Webpack/Rspack configs must be ESM[^2]. |
| — | 2026-05-20 | Last push as of this writing; MIT, ~4.3k stars. |

## References

[^1]: README migration guide and project history, Anthony Fu — `unplugin/unplugin-vue-components`. https://github.com/unplugin/unplugin-vue-components#migrate-from-vite-plugin-components
[^2]: README bundler notes: "unplugin-vue-components removed support for CommonJS after version 29.1.0." https://github.com/unplugin/unplugin-vue-components#installation
[^3]: README "TypeScript" section — `components.d.ts` augments Vue's global component interface. https://github.com/unplugin/unplugin-vue-components#typescript
[^4]: README: "We no longer accept new resolvers." https://github.com/unplugin/unplugin-vue-components/blob/main/src/core/resolvers/_READ_BEFORE_CONTRIBUTE.md

## Tags

vue, auto-import, vite, webpack, unplugin, build-tooling, typescript, components, tree-shaking, developer-experience
