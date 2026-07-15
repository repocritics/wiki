# squidfunk/mkdocs-material

> A Material Design theme and documentation framework built on top of MkDocs — Markdown in, static HTML out.

[GitHub repo](https://github.com/squidfunk/mkdocs-material) ·
[Official website](https://squidfunk.github.io/mkdocs-material/) ·
[License: MIT](https://github.com/squidfunk/mkdocs-material/blob/master/LICENSE)

## Overview

Material for MkDocs is a theme for [MkDocs](https://www.mkdocs.org/), the Python static-site generator for documentation. In practice it has outgrown the word "theme": alongside the Material Design styling it ships client-side search, instant (SPA-like) navigation, a blog plugin, social-card generation, tag indexing, versioning hooks, and a large set of Markdown extensions. It is authored primarily by Martin Donath (`@squidfunk`) and has been in continuous development since 2016[^1]. As of 2026 it is the most widely adopted MkDocs theme, used by projects including FastAPI, Pydantic, Ruff/uv (Astral), Kubernetes subprojects, and internal docs at Google, AWS, Microsoft, and others.

The defining tension is the **Insiders / sponsorware model**[^2]. New features often land first in "Insiders", a private fork available to GitHub Sponsors, and graduate into the open-source MIT edition once cumulative sponsorship crosses published funding goals. This funds a single-maintainer project sustainably, but it means the newest capabilities are time-gated behind sponsorship, and documentation frequently marks a feature as "Insiders only" — a friction point when evaluating what the free edition actually does today.

The second thing to understand is the dependency stack. Material sits on MkDocs (Jinja2 templates, a YAML config, a build pipeline) and leans heavily on **PyMdown Extensions**[^3] (also maintained by squidfunk) for admonitions, tabbed content, code highlighting, and more. You are adopting that whole chain, not a standalone tool.

## Getting Started

```bash
pip install mkdocs-material
mkdocs new .
```

Point MkDocs at the theme in `mkdocs.yml`:

```yaml
site_name: My Project
theme:
  name: material
  features:
    - navigation.instant   # SPA-style page loads
    - navigation.sections
    - content.code.copy    # copy button on code blocks
  palette:
    - scheme: default
      toggle: { icon: material/brightness-7, name: dark mode }
    - scheme: slate
      toggle: { icon: material/brightness-4, name: light mode }

markdown_extensions:
  - admonition
  - pymdownx.highlight
  - pymdownx.superfences   # nested/tabbed code fences

plugins:
  - search
```

```bash
mkdocs serve     # live-reload dev server on :8000
mkdocs build     # static HTML → ./site
```

## Architecture / How It Works

The build is MkDocs': Markdown files under `docs/` are parsed by Python-Markdown (plus enabled extensions), rendered through Jinja2 templates that Material provides, and written as static HTML to `site/`. There is no server component and no runtime — the output is deployable to any static host or GitHub Pages.

Notable internals:

- **Client-side search.** At build time a JSON search index (`search_index.json`) is generated and shipped to the browser; queries run entirely client-side (lunr-derived). No backend needed, but the index is downloaded in full, which matters at scale (see Production Notes).
- **Instant loading.** With `navigation.instant`, internal navigation is intercepted and pages are swapped via XHR, turning the site into a single-page app without full reloads. It requires the site be served from a consistent base and can interact badly with hand-written inline scripts that expect a fresh document per page.
- **Bundled front-end.** The theme's CSS/JS is a compiled TypeScript + SCSS bundle (built with Node tooling in the repo), not something you edit directly; customization goes through CSS overrides, template `overrides/`, and configuration.
- **First-party plugins.** `blog`, `tags`, `social` (social cards), `offline`, `privacy` (self-hosts external assets like fonts), and `optimize` (image compression) ship with the package or Insiders. Several are Insiders-gated.
- **Extension surface.** Much of the "Material feel" (admonitions, content tabs, annotations, Mermaid diagrams, keys) is PyMdown Extensions configuration in `mkdocs.yml`, not theme code.

The coupling story: theme, extension pack, and plugin set are co-developed by the same maintainer, so upgrades are usually coherent — but a `mkdocs-material` bump can pull in a `pymdown-extensions` requirement change, and third-party MkDocs plugins that touch templates or the nav can break across major versions.

## Production Notes

- **Search index size.** Because search is client-side, a large documentation set produces a large `search_index.json` that every visitor downloads. Very large sites (thousands of pages) should enable `search.pruning`/separate-index options or reduce indexed content; otherwise first-search latency and payload grow noticeably.
- **Social cards need native imaging libraries.** The `social` plugin renders Open Graph images and depends on Cairo/Pango (via `cairosvg`/Pillow). This is the most common CI failure: it works locally on a dev machine that already has the libraries and fails in a clean container until you install the system packages (or the maintainer's documented dependencies).
- **Insiders gating surprises.** Copy a config snippet from a blog post or another project and a feature may silently do nothing because it requires Insiders. Check the docs' "Insiders" badges before assuming a feature is available in the open-source edition.
- **`navigation.instant` edge cases.** Pages relying on per-load inline `<script>` execution, third-party embeds, or analytics that hook `DOMContentLoaded` may misbehave, since the document is not reloaded. Material documents the `instant` events to re-run such code.
- **Pin the version in CI.** `pip install mkdocs-material` unpinned means a docs build can change appearance or break on a new major release. Pin `mkdocs-material==X.Y.Z` (and pin `mkdocs` / `pymdown-extensions`) in `requirements.txt` for reproducible builds.
- **Versioned docs are external.** Multi-version documentation (e.g. `latest` vs `1.x`) is handled by `mike`, a separate tool, not by Material itself. Expect to wire it up yourself.

## When to Use / When Not

**Use when:**
- You want good-looking, searchable project docs from Markdown with minimal front-end work.
- Your project is Python-adjacent or you already live in the pip/YAML world.
- You need static output deployable to GitHub Pages or any CDN with no backend.
- You value a single, coherent, actively maintained theme over assembling your own.

**Avoid when:**
- You need rich API autodoc and cross-referencing for a large codebase — Sphinx's reStructuredText tooling is stronger there.
- You want React/MDX interactivity and components embedded in docs — Docusaurus fits better.
- You object to the sponsorware model and need every feature to be open-source-available on day one.
- Your site is enormous and client-side search payload is a hard constraint.

## Alternatives

- squidfunk/mkdocs-material vs mkdocs/mkdocs — use plain MkDocs with a lighter theme when you want a minimal footprint and no Material-specific dependencies.
- facebook/docusaurus — use instead when you need MDX, React components, and first-class i18n/versioning in a JS ecosystem.
- vuejs/vitepress — use instead for very large sites where Vite build speed and a Vue/JS toolchain are preferred.
- sphinx-doc/sphinx — use instead when reStructuredText, Python autodoc, and rich cross-references matter more than styling.
- withastro/starlight — use instead when docs sit alongside a marketing site and you want Astro's islands architecture.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2016-03 | Initial Material Design theme for MkDocs[^1]. |
| 5.0 | 2020-02 | Complete rewrite — new TypeScript/SCSS build, instant loading, refreshed architecture[^4]. |
| 7.0 | 2021 | Icons/emoji overhaul, expanded configuration. |
| 8.0 | 2021 | Continued feature growth (annotations, code features). |
| 9.0 | 2023-01 | Major release; dependency/config modernization, blog plugin maturation. |

## References

[^1]: Material for MkDocs repository and changelog. https://github.com/squidfunk/mkdocs-material
[^2]: "Insiders" — sponsorware funding and feature-graduation model. https://squidfunk.github.io/mkdocs-material/insiders/
[^3]: PyMdown Extensions, the Markdown extension pack Material depends on. https://facelessuser.github.io/pymdown-extensions/
[^4]: Material for MkDocs changelog / release history. https://squidfunk.github.io/mkdocs-material/changelog/

## Tags

python, documentation, static-site-generator, mkdocs, theme, material-design, markdown, technical-writing, jinja2, sponsorware
