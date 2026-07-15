# vuejs/vitepress

> A Vite- and Vue-powered static site generator aimed squarely at documentation sites, not general web apps.

[GitHub repo](https://github.com/vuejs/vitepress) ·
[Official website](https://vitepress.dev) ·
[License: MIT](https://github.com/vuejs/vitepress/blob/main/LICENSE)

## Overview

VitePress is the Vue team's documentation-focused static site generator, built on top of Vite and maintained under the `vuejs` org[^1]. It is the spiritual successor to VuePress[^2] — the earlier, webpack-based tool — and exists mainly because VuePress's dev server and build times became untenable as Vite matured. Where VuePress rebuilt the whole site on every change, VitePress inherits Vite's on-demand ESM dev server and gets near-instant HMR on the page you are editing.

The defining tradeoff is scope. VitePress is deliberately narrow: it ships one opinionated default theme designed for technical documentation (top nav, sidebar, on-page outline, local search) and a Markdown pipeline that lets you drop Vue components inline. That focus is why it is pleasant for docs and awkward for anything else — a marketing landing page, a blog with heavy custom layouts, or a large content app will fight the default theme and eventually push you into writing a custom theme, at which point much of the "batteries included" value evaporates.

It reached a stable 1.0 in 2024 after an unusually long alpha/beta period[^5]; for years the recommended way to use it was to pin an alpha tag. With ~18k stars and active weekly commits as of mid-2026, it is the default docs tool across the Vue ecosystem (Vue, Vite, Pinia, Vitest and most Vue-org projects use it) and a common choice for framework-agnostic docs.

## Getting Started

```bash
npm add -D vitepress
npx vitepress init          # scaffolds .vitepress/config + starter pages
npm run docs:dev            # Vite dev server with HMR
```

```ts
// .vitepress/config.ts
import { defineConfig } from 'vitepress'

export default defineConfig({
  title: 'My Docs',
  description: 'Built with VitePress',
  themeConfig: {
    nav: [{ text: 'Guide', link: '/guide/' }],
    sidebar: [
      { text: 'Introduction', items: [
        { text: 'Getting Started', link: '/guide/' },
        { text: 'Configuration', link: '/guide/config' },
      ] },
    ],
    search: { provider: 'local' },
  },
})
```

Markdown pages support frontmatter, container blocks (`::: tip`), code groups, and inline Vue:

```md
---
title: Home
---

# Hello

<script setup>
import Counter from './Counter.vue'
</script>

<Counter />
```

## Architecture / How It Works

VitePress is an SSG + SPA hybrid. At build time each Markdown file is parsed by `markdown-it`, compiled into a Vue component, and server-rendered to static HTML — so every page is crawlable and works without JavaScript. In the browser, that HTML is hydrated into a Vue 3 app, and subsequent navigation happens client-side as an SPA (route prefetching on link hover). This is the core mental model: pages are Markdown, but they are also Vue components, and both facts leak into how you write them.

Key internals:

- **Markdown = Vue templates.** After markdown-it runs, the output is treated as a Vue template. That is what makes `<Counter />` work — but it also means `{{ }}` is interpreted as Vue interpolation and `<...>` as HTML/components. This coupling is the source of most day-one confusion (see Production Notes).
- **Syntax highlighting via Shiki**[^4] — accurate, TextMate-grammar-based, run entirely at build time. It produces themed static markup with no client-side highlighter, at the cost of build time on code-heavy sites.
- **Vite for everything else.** Dev server, HMR, and the production build (Rollup under the hood) are Vite's. VitePress does not maintain its own bundler; it rides Vite's major versions, which is both a strength (fast, modern) and a coupling risk on Vite bumps.
- **Default theme** is a separate, replaceable layer exposing slots and CSS variables. You extend it (`.vitepress/theme/index.ts`) for small changes, or supply a fully custom theme for structural ones.
- **MPA mode** can disable client-side JS entirely, trading interactivity and SPA nav for smaller payloads.

## Production Notes

- **The Markdown-is-Vue footgun.** Literal `{{`, unescaped `<`, or code that looks like a Vue expression will be parsed, not printed. Fixes are `v-pre`, escaping, or fenced code blocks. This trips up nearly every new user and is easy to miss in prose that happens to contain braces.
- **Hydration mismatches.** Because pages are server-rendered then hydrated, touching `window`/`document` or browser-only libraries during render throws hydration errors. Guard with the built-in `<ClientOnly>` wrapper or defer to `onMounted`. Third-party components that assume a browser must be dynamically imported client-side.
- **Dead-link strictness.** By default VitePress fails the build on dead relative links. This is good hygiene but will break CI when you rename a page and miss a reference; there is a config escape hatch (`ignoreDeadLinks`) that teams reach for too quickly.
- **Build time scales with code blocks.** Shiki highlighting is build-time; very large docs with thousands of code samples see noticeably longer builds. There is no incremental build — a change rebuilds the whole site (dev HMR is fine; `docs:build` is not incremental).
- **Local search has limits.** The built-in local search (MiniSearch-based) ships the index to the client and is fine for small-to-mid docs; large sites should move to Algolia DocSearch to avoid shipping a large index.
- **Vite/Vue version churn.** Because VitePress tracks Vite closely, a Vite major bump can require a VitePress upgrade in lockstep, and community Markdown/Vite plugins occasionally lag. Pin versions in CI.
- **Not a CMS or blog engine.** There is no built-in content collection, tagging, or pagination system; a blog is doable via frontmatter + `createContentLoader`, but you are assembling it yourself.

## When to Use / When Not

**Use when:**
- You are documenting a library, framework, or API and want a fast, conventional docs site.
- You are already in the Vue/Vite ecosystem and want inline Vue components in your docs.
- You want static output deployable to any host (GitHub Pages, Netlify, Cloudflare Pages) with zero server.

**Avoid when:**
- You need a general marketing site, a full blog platform, or heavily custom page layouts — the default theme resists this and a custom theme is real work.
- You do not use Vue and do not want Vue's rendering model and its hydration constraints leaking into your Markdown.
- You need incremental builds over a very large content set, or a mature plugin/versioning system out of the box.

## Alternatives

- vuepress/vuepress — the webpack-based predecessor; still maintained but slower and largely superseded by VitePress for new projects.
- facebook/docusaurus — use instead when you want React, built-in docs versioning, and a richer plugin ecosystem, and don't mind a heavier build.
- withastro/starlight — use instead when you want a framework-agnostic docs theme on Astro's islands model rather than committing to Vue.
- squidfunk/mkdocs-material — use instead when your team lives in Python/Markdown and does not need a JS component runtime.
- rust-lang/mdBook — use instead for simple, fast static books where interactivity and components are not needed.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2020-04-30 | Public repo under `vuejs`, positioned as VuePress successor[^1]. |
| 1.0.0-alpha | 2022 | Long-running alpha; recommended usage was pinning alpha tags. |
| 1.0.0 | 2024-03 | First stable release after ~2 years of alpha/beta[^5]. |
| 1.x | 2024–2026 | Ongoing releases tracking Vite/Vue majors; local search, i18n, Shiki updates. |

## References

[^1]: VitePress documentation and homepage. https://vitepress.dev
[^2]: VuePress — the earlier webpack-based Vue SSG. https://vuepress.vuejs.org
[^3]: Vite — the underlying dev server and build tool. https://vite.dev
[^4]: Shiki — the build-time syntax highlighter used by VitePress. https://shiki.style
[^5]: VitePress CHANGELOG. https://github.com/vuejs/vitepress/blob/main/CHANGELOG.md
[^6]: "Using Vue in Markdown" — VitePress guide. https://vitepress.dev/guide/using-vue

## Tags

typescript, vue, vite, static-site-generator, documentation, markdown, ssg, docs-generator, frontend, jamstack
