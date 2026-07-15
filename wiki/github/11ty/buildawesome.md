# 11ty/buildawesome

> A JavaScript static site generator that transforms a directory of templates into HTML and ships zero client-side JavaScript by default. Known for most of its life as Eleventy.

[GitHub repo](https://github.com/11ty/buildawesome) ·
[Official website](https://www.11ty.dev/) ·
[License: MIT](https://github.com/11ty/buildawesome/blob/main/LICENSE)

## Overview

Eleventy (created by Zach Leatherman, first released 2018) is a static site generator written in JavaScript, positioned explicitly as "a simpler alternative to Jekyll"[^1]. It takes a directory of templates — Markdown, Nunjucks, Liquid, HTML, plain JavaScript, WebC, and others — and produces a folder of static HTML. Its defining stance is subtractive: unlike Gatsby or Next.js, Eleventy does not bundle a client-side framework, does not hydrate, and emits no JavaScript to the browser unless you write it yourself. What you author is what ships.

The project's central abstraction is the **data cascade**: page output is the merge of front matter, directory-level data files, global `_data`, layout data, and computed data, resolved by a documented precedence order[^2]. Most of Eleventy's power (collections, pagination, permalink control) is expressed through this cascade rather than through a plugin runtime. The tradeoff is conceptual: the cascade is flexible but has enough merge rules that debugging "where did this value come from" is a recurring beginner pain point.

The GitHub repository and npm package were renamed in a 2026 rebrand — the repo is now `11ty/buildawesome` and the package publishes under the `@awesome.me` scope (Fonticons/Font Awesome's namespace, where Leatherman works), with `@11ty/eleventy` retained as a backwards-compatible alias[^3]. The `11ty` branding, `.11ty.dev` docs, and existing installs continue to work; treat "Eleventy" and "Build Awesome" as the same project.

## Getting Started

```bash
npm install @11ty/eleventy --save-dev      # alias still works
# or the new scope:
npm install @awesome.me/buildawesome --save-dev
npx @11ty/eleventy --serve                 # dev server with hot reload
```

```js
// eleventy.config.js  (ESM, Eleventy 3.x)
export default function (eleventyConfig) {
  eleventyConfig.addPassthroughCopy("src/css");
  return { dir: { input: "src", output: "_site" } };
}
```

```njk
{# src/index.njk #}
---
title: Hello
layout: base.njk
---
<h1>{{ title }}</h1>
```

## Architecture / How It Works

Eleventy is a build orchestrator over a set of pluggable template engines, not a rendering engine itself. The pipeline:

1. **Discovery** — walk the input directory, classify each file by extension into a template engine (`.md`, `.njk`, `.liquid`, `.11ty.js`, `.webc`, …).
2. **Data cascade** — for each template, merge front matter + `*.11tydata.js`/JSON directory data + global `_data/**` + computed data, in a fixed precedence[^2].
3. **Collections** — templates tagged via `tags` front matter are grouped into arrays available to every template, enabling blog-index and feed patterns without a database.
4. **Render** — each engine renders to HTML; permalinks and pagination determine the output path(s). A single source template can fan out to many pages via `pagination`.
5. **Passthrough + transforms** — static assets copied verbatim, HTML post-processed by registered transforms.

Everything author-facing is registered on the config object: `addFilter`, `addShortcode`, `addCollection`, `addTransform`, `addPlugin`. There is no virtual DOM, no hydration boundary, and no server runtime — the output is a static directory. Interactivity is entirely the author's responsibility (hand-written `<script>`, or a plugin like the bundled WebC for single-file components).

Template engines are provided by third-party libraries (LiquidJS, Nunjucks, `markdown-it`), which means Eleventy inherits their quirks and their release cadence. This keeps Eleventy's own surface small but means "how do I do X in a loop" often resolves to the underlying engine's docs, not Eleventy's.

## Production Notes

**Nunjucks/Liquid are the coupling risk, not Eleventy.** Because rendering is delegated, subtle behavior (whitespace control, async filter support, autoescaping defaults) comes from LiquidJS or Nunjucks. Nunjucks in particular is only lightly maintained upstream; teams standardizing on it should be aware they are depending on a slow-moving library.

**Build performance is good but not Hugo-class.** Eleventy is fast for the typical blog/docs site (hundreds to low-thousands of pages) and incremental builds (`--incremental`) help during development. At tens of thousands of pages the JavaScript runtime and per-template data-cascade merging make it materially slower than Go-based Hugo. Choose accordingly for very large content sets.

**The 3.0 ESM migration is the sharpest upgrade edge.** Eleventy 3.0 moved the project to ES modules and can now be configured with ESM (`eleventy.config.js` using `export default`)[^4]. Projects with CommonJS config files, `require()`-based data files, or CJS-only plugins need migration work. Config filename also shifted from `.eleventy.js` toward `eleventy.config.js` (both still resolve).

**Data cascade debugging.** The most common production confusion is precedence: a value set in front matter overriding — or being overridden by — directory data or computed data in a way that surprises the author. `eleventyComputed` runs late and can silently shadow earlier values. When output is wrong, inspect the merged data (e.g. dump the data object to a debug template) before suspecting the template.

**No official image/asset pipeline in core.** Image optimization is a separate plugin (`@11ty/eleventy-img`); CSS/JS bundling is left to the author or a bundler plugin. This is deliberate minimalism but means a production site assembles several plugins rather than getting an integrated asset story.

**Deployment is trivial by design** — the `_site` output is plain files, hostable on any static host or CDN (Netlify, Cloudflare Pages, GitHub Pages, S3) with no server component and no vendor lock-in.

## When to Use / When Not

**Use when:**
- You want a content site (blog, docs, marketing) that ships as pure static HTML with no framework runtime.
- You value zero client JavaScript by default and want to add interactivity deliberately.
- You want to keep authoring in Markdown/Nunjucks/Liquid and not learn a component framework.
- You want simple, host-anywhere output with no vendor coupling.

**Avoid when:**
- You need component-level client interactivity/hydration out of the box — Astro's islands or a React framework fit better.
- Your site has tens of thousands of pages and build time dominates — Hugo is faster at that scale.
- Your team wants an integrated, batteries-included asset/image/data pipeline rather than assembling plugins.
- You want to author in JSX/React components specifically.

## Alternatives

- withastro/astro — component-based content sites with optional islands hydration; use when you need interactive UI components, not just static HTML.
- gohugoio/hugo — Go single-binary SSG; use when you have very large sites and build speed is the priority.
- jekyll/jekyll — the Ruby predecessor Eleventy was designed to replace; use when you are on GitHub Pages' native build or invested in Ruby.
- getzola/zola — Rust single-binary SSG; use when you want zero Node/JS dependency and one executable.
- gatsbyjs/gatsby — React-based SSG; use when you specifically want a React component model (though its momentum has faded).

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2018-01 | First public release; multi-engine static generation[^1]. |
| 1.0.0 | 2022-02 | First stable 1.x; serverless/Edge experiments, config API maturity. |
| 2.0.0 | 2023-02 | Improved i18n, `eleventyConfig` refinements, better error output. |
| 3.0.0 | 2024-10 | ESM support, WebC and bundle plugins, Node ESM config[^4]. |
| — | 2026 | Repo/npm rebrand to `buildawesome` / `@awesome.me` scope[^3]. |

## References

[^1]: Eleventy README and docs — "A simpler site generator... an alternative to Jekyll." https://www.11ty.dev/docs/
[^2]: Eleventy docs, "The Data Cascade." https://www.11ty.dev/docs/data-cascade/
[^3]: GitHub API `repos/11ty/eleventy` resolves to `full_name: 11ty/buildawesome`; README documents `npm install @awesome.me/buildawesome` with `@11ty/eleventy` retained as backwards-compatible. https://github.com/11ty/buildawesome
[^4]: Eleventy docs, "Get Started / Configuration" (ESM support in 3.0). https://www.11ty.dev/docs/config/

## Tags

javascript, static-site-generator, ssg, jamstack, markdown, nunjucks, templating, blog-engine, documentation-tool, zero-js, eleventy
