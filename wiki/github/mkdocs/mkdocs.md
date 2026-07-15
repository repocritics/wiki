# mkdocs/mkdocs

> A Python static site generator that turns a folder of Markdown files and one YAML config into a documentation website.

[GitHub repo](https://github.com/mkdocs/mkdocs) ·
[Official website](https://www.mkdocs.org) ·
[License: BSD-2-Clause](https://github.com/mkdocs/mkdocs/blob/master/LICENSE)

## Overview

MkDocs is a static site generator specialized for project documentation. Source pages are plain Markdown, the site is described by a single `mkdocs.yml` file, and the output is a directory of static HTML that can be served from anywhere — GitHub Pages, S3, Netlify, or a plain web server[^1]. It was created by Tom Christie (also the author of Django REST Framework and the `encode` HTTP stack) in 2014, and is now maintained by a small volunteer team.

The defining characteristic of MkDocs is deliberate minimalism. The core does three things: read config, run Markdown through Python-Markdown[^2], and render the result through Jinja2 theme templates. Everything beyond that — admonitions, tabbed content, versioning, API-doc autogeneration, search tuning — lives in themes, Markdown extensions, and plugins. This is the project's defining tension: the base tool is small and easy to reason about, but a real-world MkDocs site is almost always *MkDocs core + the Material theme + a handful of plugins*, and most of the features people attribute to "MkDocs" actually belong to `squidfunk/mkdocs-material`[^3].

MkDocs targets developers who want documentation to live next to code in the repository, be reviewed in pull requests, and build in CI. It does not target general-purpose sites, blogs, or content that needs a database. If your docs are Markdown and your audience is developers, MkDocs is one of the two or three default choices; if you need reStructuredText, deep cross-referencing, or Python API autodoc, Sphinx is the more common pick.

## Getting Started

```bash
pip install mkdocs
mkdocs new my-docs
cd my-docs
mkdocs serve       # dev server with live reload at http://127.0.0.1:8000
```

A minimal `mkdocs.yml`:

```yaml
site_name: My Project
nav:
  - Home: index.md
  - Guide: guide.md
theme:
  name: material    # requires: pip install mkdocs-material
plugins:
  - search
markdown_extensions:
  - admonition
  - toc:
      permalink: true
```

Build and deploy:

```bash
mkdocs build              # renders to ./site/
mkdocs gh-deploy          # builds and force-pushes ./site to the gh-pages branch
```

`mkdocs gh-deploy` commits the built site to `gh-pages` and pushes it — convenient, but it rewrites that branch's history, so it should point at a branch you do not otherwise touch[^1].

## Architecture / How It Works

The build pipeline is linear and fully static — there is no runtime, no server-side rendering, no client hydration:

1. **Config** — `mkdocs.yml` is parsed and validated against a schema. `theme`, `plugins`, and `markdown_extensions` are resolved to installed Python packages via entry points. A missing plugin or theme is a hard error at this stage.
2. **File collection** — the `docs_dir` (default `docs/`) is walked; `.md` files become pages, everything else is copied as a static asset.
3. **Navigation** — the `nav:` tree in config defines page order and titles. If `nav` is omitted, MkDocs infers a flat structure from the file tree (alphabetical), which is why non-trivial sites always declare `nav` explicitly.
4. **Markdown → HTML** — each page runs through Python-Markdown with the configured extensions.
5. **Template render** — the HTML is injected into the theme's Jinja2 templates and written to `site_dir` (default `site/`).

**Plugins** hook into this pipeline through a documented event system[^4] — `on_config`, `on_files`, `on_nav`, `on_page_markdown`, `on_page_content`, `on_post_build`, and others. A plugin is a Python class registered via a `mkdocs.plugins` entry point; it can rewrite content, inject files, or emit extra artifacts. The built-in `search` plugin is itself implemented this way, building a client-side Lunr.js index at build time.

A critical detail: **MkDocs uses Python-Markdown, not CommonMark**. Nested list indentation, code fences, and some inline edge cases behave differently from GitHub-Flavored Markdown, which surprises authors who assume their README rendering will match[^2]. The `pymdownx` extension pack (part of the Material ecosystem) closes most of the gap but must be enabled explicitly.

## Production Notes

**No incremental builds.** `mkdocs build` rebuilds the entire site every time; there is no per-page caching between runs. For small-to-medium docs this is instant, but sites with thousands of pages (large monorepo docs, generated API references) can take tens of seconds to minutes per build. `mkdocs serve` rebuilds the whole site on every file change — live reload feels fast only because most sites are small.

**Search index scales poorly.** The default search builds a single Lunr.js index that ships to the browser and is parsed client-side. On large sites this index grows into megabytes and hurts first-load performance; teams commonly switch to the Material theme's search or an external service (Algolia DocSearch) at scale.

**Versioning is not built in.** MkDocs has no concept of documentation versions. The community standard is the `mike` plugin, which manages multiple built versions on the `gh-pages` branch. Retrofitting versioning onto an existing site is a known migration headache.

**Navigation is manual.** The `nav` tree is hand-maintained YAML. Large sites use `mkdocs-awesome-pages-plugin` or `mkdocs-literate-nav` to avoid editing one central list on every page addition. Without them, a new page that is not added to `nav` still builds but is unreachable except by direct URL.

**The theme is the product.** The bundled `mkdocs` and `readthedocs` themes are functional but plain. Nearly every serious site uses Material for MkDocs, which is a separate package with its own release cadence, its own config surface, and (as of recent years) a sponsor-gated "Insiders" edition where some features land before the open version[^3]. Pinning both `mkdocs` and `mkdocs-material` versions in CI is standard practice, because Material occasionally requires a newer MkDocs core.

**Strict mode for CI.** `mkdocs build --strict` turns warnings (broken internal links, missing nav references, orphaned files) into build failures. It is off by default; enabling it in CI is the single most effective quality gate for a docs repo.

## When to Use / When Not

**Use when:**
- Your documentation is Markdown and lives in the same repo as the code.
- You want docs reviewed in pull requests and built in CI as static files.
- You want a fast path to a searchable, navigable site with the Material theme.
- You are deploying to GitHub Pages or any static host and do not want a build server.

**Avoid when:**
- You need reStructuredText, deep cross-references, or Python API autodoc — Sphinx is the mature choice (though `mkdocstrings` narrows this gap for MkDocs).
- You want React/MDX components and interactive docs — Docusaurus fits better.
- Your site is a blog or marketing site, not developer docs — a general SSG (Hugo, Astro) is a better fit.
- You have thousands of pages and need incremental builds and versioned output out of the box.

## Alternatives

- sphinx-doc/sphinx — reStructuredText (and now Markdown via MyST), the standard for Python API documentation; use instead when you need autodoc and rich cross-referencing.
- facebook/docusaurus — React/MDX-based; use instead when docs need interactive components or a versioned React site.
- gohugoio/hugo — general-purpose, extremely fast static generator; use instead when docs are one part of a larger content site.
- vuejs/vitepress — Vue-powered docs SSG; use instead when the team is in the Vue/Vite ecosystem and wants fast client-side navigation.
- rust-lang/mdBook — minimal, single-binary book generator; use instead when you want zero Python dependency and a simpler book format.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2014 | Initial public release by Tom Christie[^1]. |
| 1.0 | 2018-08 | First stable release; plugin API and `nav` config stabilized[^5]. |
| 1.1 | 2020 | Theme and search improvements; Python 2 support dropped. |
| 1.2 | 2021 | Faster dev server, dirty-reload improvements. |
| 1.4 | 2022 | Config-schema rework, theme hooks. |
| 1.5 | 2023 | Redirect/link validation, `--strict` link checking improvements[^5]. |
| 1.6 | 2024 | Newer Python support, build and validation refinements. |

## References

[^1]: MkDocs — official site and user guide. https://www.mkdocs.org
[^2]: Python-Markdown — the Markdown parser MkDocs uses. https://python-markdown.github.io
[^3]: Material for MkDocs — the de facto standard theme and plugin suite. https://squidfunk.github.io/mkdocs-material/
[^4]: MkDocs plugin developer guide (event hooks). https://www.mkdocs.org/dev-guide/plugins/
[^5]: MkDocs release notes / changelog. https://www.mkdocs.org/about/release-notes/

## Tags

python, documentation, static-site-generator, markdown, docs-as-code, jinja2, yaml, developer-tools, github-pages, site-generator
