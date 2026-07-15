# docsifyjs/docsify

> A runtime documentation site generator that renders Markdown to a single-page app in the browser — no build step, no generated HTML.

[GitHub repo](https://github.com/docsifyjs/docsify) ·
[Official website](https://docsify.js.org) ·
[License: MIT](https://github.com/docsifyjs/docsify/blob/develop/LICENSE)

## Overview

Docsify, created by Qingwei Li (@QingWei-Li) and first released in 2016[^1], is a
documentation site generator that does its work at runtime in the browser rather
than at build time. You ship a single `index.html`, a script tag pointing at the
docsify bundle, and a folder of Markdown files. When a visitor loads a page,
docsify reads the URL, fetches the corresponding `.md` file over HTTP, parses it,
and injects the resulting HTML into the page. There is no static build, no
compiled output directory, and nothing to deploy beyond the raw Markdown.

That single design choice is the whole story of the project. It makes docsify the
lowest-friction option in the documentation-tooling space: clone a template, write
Markdown, push to GitHub Pages, done. It is popular for READMEs promoted to full
sites, internal/team docs, and open-source project pages — the ~31.4k stars and
~5.8k forks reflect a decade of that niche being genuinely well-served. The cost
of the same choice is that content only exists after JavaScript runs, which
trades away static HTML, and with it a large part of the SEO and
no-JS-fallback story. Docsify is best understood not as a competitor to static
generators but as a different point on the build-vs-runtime axis.

The project is on the long-lived v4 line and remains maintained (commits through
mid-2026); the default branch is `develop`, where next-major work lands.

## Getting Started

No install is required for the site itself. The docsify-cli helps scaffold and
serve locally:

```bash
npm i -g docsify-cli
docsify init ./docs      # writes index.html, README.md, .nojekyll
docsify serve ./docs     # local preview with live reload
```

A minimal hand-written `index.html` is the entire runtime:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8" />
  <link rel="stylesheet" href="//cdn.jsdelivr.net/npm/docsify@4/lib/themes/vue.css" />
</head>
<body>
  <div id="app"></div>
  <script>
    window.$docsify = {
      name: 'My Docs',
      loadSidebar: true,      // read _sidebar.md
      search: 'auto',         // client-side full-text search
    };
  </script>
  <script src="//cdn.jsdelivr.net/npm/docsify@4"></script>
  <script src="//cdn.jsdelivr.net/npm/docsify/lib/plugins/search.min.js"></script>
</body>
</html>
```

Content lives in `README.md` (the home route), plus optional `_sidebar.md`,
`_navbar.md`, `_coverpage.md`, and a `.nojekyll` file so GitHub Pages does not
strip `_`-prefixed files.

## Architecture / How It Works

Docsify is a client-side single-page application. The lifecycle on each
navigation:

1. **Route resolution** — the router maps the URL to a Markdown file path. Default
   mode is hash routing (`example.com/#/guide/install`), which needs no server
   configuration because the path after `#` never hits the server. An optional
   `history` mode gives clean URLs but requires the host to rewrite unknown paths
   back to `index.html`.
2. **Fetch** — the target `.md` file is loaded via `fetch`/XHR. Files are pulled
   individually and on demand; there is no bundle of all pages.
3. **Compile** — Markdown is parsed (marked) into HTML, with syntax highlighting
   by Prism. Hooks (`beforeEach`, `afterEach`, `doneEach`, `mounted`, `ready`)
   let plugins transform Markdown or the rendered DOM at each stage.
4. **Render** — HTML is injected into `#app`. The sidebar is either generated from
   the page's headings or read from `_sidebar.md`.

Everything is configured through the global `window.$docsify` object — there is no
config file format, no schema, and (historically) no first-class TypeScript types
for it. Plugins are just functions pushed onto `$docsify.plugins` that register on
the lifecycle hooks; the official plugins (full-text search, pagination,
copy-to-clipboard, emoji, Google Analytics, edit-on-GitHub) follow this pattern,
as does the third-party ecosystem catalogued in awesome-docsify.

The **full-text search plugin** is worth understanding because it shapes
performance: it has no server index, so to build one it fetches every Markdown
file listed in the sidebar, tokenizes them in the browser, and caches the index
in `localStorage`. Search quality and first-search latency are therefore a
function of how many pages you have.

## Production Notes

**SEO and crawlers are the defining caveat.** Because pages are assembled by
JavaScript after load, the initial HTML response contains no article content.
Google's renderer can execute the JS, but coverage is not guaranteed and other
crawlers, link-preview/Open Graph unfurlers, and no-JS clients see an empty
shell — per-page `<title>`/`<meta>` are also not present in the source HTML.
Mitigations exist but are all workarounds: a prerender service (Prerender.io and
similar) that serves rendered HTML to bots, `docsify-ssr`-style approaches, or
accepting that discoverability will trail a static generator. If organic search
matters, this is usually the reason teams pick something else.

**Many small requests.** No build means no concatenation: each visited page is a
separate network round-trip, and enabling search fetches *all* pages up front to
index them. On large docs sites this is noticeable on cold load and on slow
connections. There is no code-splitting story to tune — the lever is "fewer,
larger pages" or a CDN in front.

**CDN and version pinning.** The canonical setup loads docsify from jsDelivr/unpkg.
Pin the major (`docsify@4`) rather than `docsify` (floating latest); an unpinned
tag can pull a breaking change without warning, and a CDN outage takes the whole
site down. Self-hosting the assets removes both risks and is supported.

**Loading flash and empty-state.** Until the script loads and the first fetch
completes, the user sees a blank/placeholder page. Configure the loading
indicator and be aware that any JS error aborts rendering entirely — a broken
plugin yields a blank site, not a degraded one.

**Relative links and nesting.** Links between Markdown files and relative asset
paths are a recurring source of confusion, especially in `history` mode and under
sub-path deployments; `relativePath`, `basePath`, and `alias` config exist
precisely because the naive cases break. `.nojekyll` is mandatory on GitHub Pages
or `_sidebar.md`/`_media/` are silently 404'd.

## When to Use / When Not

**Use when:**
- You want documentation live from Markdown with zero build pipeline.
- The audience is internal/technical and reaches docs via direct links, not search.
- You're publishing to GitHub Pages and want the shortest path from README to site.
- Content changes constantly and you value edit-and-refresh over generated output.

**Avoid when:**
- Organic search / SEO is a primary channel — you need static, per-page HTML.
- The site is large enough that per-page fetches and full-corpus search indexing
  hurt load times.
- You need versioned docs, build-time i18n, MDX/components, or a typed config —
  static generators cover these first-class.
- You require guaranteed no-JS rendering (accessibility mandates, locked-down
  crawlers, air-gapped viewers).

## Alternatives

- vuejs/vitepress — Vite-based static generator, Vue-powered; use when you want
  docsify's Markdown-first feel but static HTML and real SEO.
- facebook/docusaurus — React, first-class versioning/i18n/plugins; use for large,
  long-lived docs that outgrow a single Markdown folder.
- squidfunk/mkdocs-material — Python/MkDocs, static output, strong theme; use when
  your toolchain is already Python.
- withastro/astro — with the Starlight docs theme, islands architecture and
  content collections; use for content sites where performance and SEO matter.
- slatedocs/slate — single-page API reference generator; use specifically for
  three-pane REST API docs.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2016-11 | First public release; runtime Markdown-to-SPA concept[^1]. |
| 4.0 | 2017 | v4 rewrite; the long-lived stable major line. |
| 4.x | 2018–2026 | Ongoing 4.x releases: search, themes, plugin API, a11y and build/tooling modernization[^2]. |

(Exact 4.x minor release dates are omitted where not verified; see the GitHub
releases page for the authoritative changelog.)

## References

[^1]: docsify repository and documentation, docsifyjs/docsify — created
2016-11-20. https://github.com/docsifyjs/docsify
[^2]: docsify official documentation. https://docsify.js.org

## Tags

javascript, documentation, documentation-tool, markdown, static-site-generator, spa, github-pages, client-side-rendering, docs-as-code, zero-build
