# antfu-collective/vitesse

> An opinionated Vite + Vue 3 starter template that wires together Anthony Fu's auto-import plugin ecosystem into a single scaffold.

[GitHub repo](https://github.com/antfu-collective/vitesse) ·
[Live demo](https://vitesse.netlify.app/) ·
[License: MIT](https://github.com/antfu-collective/vitesse/blob/main/LICENSE)

## Overview

Vitesse is a GitHub *template repository*, not a library. You clone it once (via
`degit` or GitHub's "Use this template"), and from that moment you own the code —
there is no `vitesse` package in your `dependencies` and no upgrade channel. It
was created in August 2020 during the early Vue 3 + Vite transition[^1] and
became the canonical reference for "how Anthony Fu (antfu) sets up a Vue app":
file-based routing, component and API auto-imports, UnoCSS, Pinia, i18n, PWA, and
SSG, all pre-configured and glued together.

The defining tension is *magic vs. legibility*. Vitesse leans hard on
unplugin-based auto-imports — you write `<RouterLink>`, `ref()`, or `useHead()`
with no import statement, and build-time plugins resolve them and generate
`.d.ts` files so TypeScript still type-checks. This makes greenfield prototyping
very fast and makes the resulting code hard to read out of context, since almost
nothing is explicitly imported. Nearly every plugin in the stack is authored or
co-maintained by antfu, so the template is also a concentrated bet on one
person's tooling ecosystem.

As of 2026 the template is in reference/maintenance mode. The README now
explicitly recommends [Nuxt](https://nuxt.com) for teams wanting a better-
supported Vue DX and warns to "expect slower updates"[^2]. The last significant
push was February 2026; it is stable but no longer rapidly evolving.

## Getting Started

Vitesse is consumed by scaffolding, not by installing. It requires Node ≥ 14.18
(in practice, current deps push this higher) and prefers pnpm.

```bash
# Clone with a clean git history
npx degit antfu-collective/vitesse my-app
cd my-app
pnpm install
pnpm dev          # dev server on http://localhost:3333
```

A page is just a `.vue` file under `src/pages/` — no route registration needed:

```vue
<!-- src/pages/hello/[name].vue  ->  route /hello/:name -->
<script setup lang="ts">
// useRoute() and ref() are auto-imported — no import lines
const route = useRoute()
const count = ref(0)
</script>

<template>
  <h1>Hi {{ route.params.name }}</h1>
  <button @click="count++">Clicked {{ count }}</button>
</template>
```

## Architecture / How It Works

Vitesse is best understood as a *curated `vite.config.ts` plus conventions*. The
heavy lifting is done by a stack of independent plugins, most from antfu's
ecosystem:

- **unplugin-vue-router** — file-system routing from `src/pages/`, generating a
  typed route table (`typed-router.d.ts`).
- **vite-plugin-vue-layouts** — wraps pages in layouts from `src/layouts/`.
- **unplugin-vue-components** — auto-imports Vue components on first use, writing
  `components.d.ts`.
- **unplugin-auto-import** — auto-imports Composition-API functions (`ref`,
  `computed`, `useRoute`, VueUse helpers), writing `auto-imports.d.ts`.
- **UnoCSS** — on-demand atomic CSS plus pure-CSS icons via `preset-icons`
  (migrated here from Windi CSS during UnoCSS's development).
- **vite-ssg** — build-time static-site generation with optional critical-CSS
  inlining (beasties). The same codebase runs as SPA in dev and pre-renders at
  build.
- **Pinia**, **Vue I18n**, **vite-plugin-pwa**, **unplugin-vue-markdown**
  (Markdown-as-component), and **@unhead/vue** round out state, i18n, offline,
  content, and `<head>` management.

The critical mechanism is the generated `.d.ts` files. Auto-imports only
type-check because the plugins emit declaration files during `vite dev`/`build`;
these are what let the editor resolve un-imported globals. This couples your IDE
correctness to the dev server having run at least once, and it means the
generated files must either be committed or regenerated in CI before
`vue-tsc`/ESLint run.

## Production Notes

**It's a fork, not a dependency — you inherit drift.** Once scaffolded, your app
never receives Vitesse updates. Any plugin bumps, security fixes, or config
improvements upstream must be reconciled by hand. Teams that scaffolded years ago
are effectively maintaining a private fork of a 2021-era plugin matrix.

**Auto-import magic has a cold-start tax.** Fresh clone, or a teammate opening
the repo without running `pnpm dev` first, will see a flood of "cannot find name
`ref`/`useRoute`" TypeScript and ESLint errors until the generated `*.d.ts`
files exist. CI must run a build (or the plugins' codegen) *before* the typecheck
step, or the pipeline fails on phantom errors.

**SSG is not free SSR.** `vite-ssg` pre-renders at build time in a Node context,
so any component reaching for `window`/`document` at setup, or a dependency that
isn't SSR-safe, breaks the build. This is the most common surprise for people who
assumed the SPA-first template "also does SSR."

**Opinionated to the point of friction.** The bundled `@antfu/eslint-config`
enforces no-semicolons and single quotes and will rewrite code aggressively on
save; the icon workflow assumes UnoCSS `preset-icons`; the router assumes
file-based conventions. Deviating means unwinding config rather than adding to it.

**Bus-factor concentration.** The value proposition is that one maintainer's
plugins are known to work together. The risk is the same: a large share of the
stack shares a single primary author, so ecosystem-wide churn (e.g., a Vite major
bump) tends to ripple through many dependencies at once.

## When to Use / When Not

**Use when:**
- You're prototyping a Vue 3 SPA/SSG and want batteries-included conventions now.
- You already like antfu's stack (UnoCSS, VueUse, auto-imports) and want it wired.
- You want a reference implementation of file routing + layouts + SSG in Vite.

**Avoid when:**
- You want a maintained *framework* with an upgrade path — use Nuxt (the author's
  own current recommendation).
- Your team dislikes implicit auto-imports or has an existing ESLint/style guide.
- You need SSR (streaming/server data), which this template does not target.
- You want minimal magic and explicit imports for long-term legibility.

## Alternatives

- vuejs/create-vue — the official Vue scaffolder; minimal, explicit imports, no auto-import magic. Use when you want a legible baseline you fully control.
- nuxt/nuxt — full Vue meta-framework with SSR/SSG and a real upgrade path. Use when you need a maintained framework rather than a one-shot template (antfu now points here).
- antfu/vitesse-lite — the stripped-down Vitesse (no SSG/i18n/markdown/PWA). Use when you want the conventions with far less surface area.
- vitejs/vite — the bare build tool. Use when you'd rather assemble your own plugin set instead of inheriting an opinionated one.
- unjs/nuxt (Nuxt UI/starter kits) — for content or dashboard sites where a framework's module ecosystem beats a template's frozen config.

## History

Vitesse has no semantic-version releases — it is a template repo updated by
continuous commits, so the timeline below is milestone-based rather than tagged.

| Milestone | Date | Notes |
|-----------|------|-------|
| Created | 2020-08-09 | Early Vue 3 + Vite transition; original SPA-first scaffold[^1]. |
| CSS engine swap | ~2021–2022 | Migrated from Windi CSS to UnoCSS as antfu's atomic-CSS engine matured. |
| Router plugin swap | ~2023 | Adopted unplugin-vue-router for typed file-based routing. |
| Moved to org | ~2024 | Repository relocated from `antfu/` to the `antfu-collective/` org. |
| Nuxt recommendation | by 2026 | README adds guidance to prefer Nuxt; slower-update notice[^2]. |
| Last significant push | 2026-02-25 | Maintenance/reference mode; ~9.4k stars, MIT. |

## References

[^1]: antfu, "Vitesse" template repository — created 2020-08-09. https://github.com/antfu-collective/vitesse
[^2]: Vitesse README "Note" block recommending Nuxt and warning of slower updates. https://github.com/antfu-collective/vitesse/blob/main/README.md
[^3]: Vitesse live demo. https://vitesse.netlify.app/
[^4]: UnoCSS — on-demand atomic CSS engine used by the template. https://github.com/unocss/unocss

## Tags

vue, vite, typescript, starter-template, scaffold, spa, ssg, unocss, pinia, auto-import, frontend, pnpm
