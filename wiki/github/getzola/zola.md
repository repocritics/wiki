# getzola/zola

> A static site generator that ships as one dependency-free binary with Sass, syntax highlighting, search, and image processing built in.

[GitHub repo](https://github.com/getzola/zola) ·
[Official website](https://www.getzola.org) ·
[License: EUPL-1.2](https://github.com/getzola/zola/blob/master/LICENSE)

## Overview

Zola is a static site generator written in Rust, originally released in 2017 under the name Gutenberg and renamed to Zola in 2018 to avoid the clash with WordPress's Gutenberg editor[^1]. It compiles Markdown content plus [Tera](https://keats.github.io/tera/) templates into a static `public/` directory. The project's explicit design goal is that everything a typical content site needs — Sass compilation, syntax highlighting, a client-side search index, image resizing, taxonomies, pagination — is compiled into the single binary, so there is no Node, Ruby, or Go toolchain and no plugin install step to manage[^2].

Its origin is openly reactive: the author built it out of dislike for Hugo's Go template engine, and chose Tera, a Jinja2-style engine also written by the same author, as the templating layer[^2]. That lineage defines Zola's character. Compared to Hugo it trades a very large theme ecosystem and Go's template flexibility for a smaller, more predictable surface and templates that most people find easier to read.

The central tradeoff is the flip side of "everything built in": Zola has no plugin or extension system. If a capability is not in the binary, you cannot add it with custom code — you work within Tera, shortcodes, and the provided built-in functions, or you preprocess content outside Zola. For the sites Zola targets (blogs, documentation, project landing pages) this is rarely a wall; for anything needing bespoke build-time logic it is a hard ceiling.

## Getting Started

```bash
# macOS
brew install zola
# from source (Rust toolchain)
cargo install zola
# or download a prebuilt binary from the releases page
```

```bash
zola init my-site      # scaffolds config.toml, content/, templates/, ...
cd my-site
zola serve             # dev server with live reload on http://127.0.0.1:1111
zola build             # outputs static site to ./public
```

```markdown
<!-- content/blog/hello.md -->
+++
title = "Hello Zola"
date = 2026-01-15
[taxonomies]
tags = ["intro"]
+++

Body written in **Markdown**. Shortcodes and internal links work here.
```

```jinja2
{# templates/page.html — Tera #}
{% extends "base.html" %}
{% block content %}
  <h1>{{ page.title }}</h1>
  {{ page.content | safe }}
{% endblock content %}
```

## Architecture / How It Works

Zola's pipeline is a straight compile: it walks `content/`, parses each file's TOML front matter (delimited by `+++`) and Markdown body, builds an in-memory tree of sections (`_index.md`) and pages, then renders each node through a Tera template selected by convention or front-matter override. Output is written to `public/`.

The Markdown parser is pulldown-cmark, so Zola tracks the CommonMark spec closely. Syntax highlighting uses syntect with embedded Sublime Text syntaxes and themes, done at build time — highlighted code is baked into the HTML with no client-side JS required. Sass/SCSS is compiled in-process. Search is an optional client-side index (elasticlunr-format JSON) that Zola generates at build; there is no server component, so search runs entirely in the browser[^2].

Templating is where most of Zola's behavior lives. Tera provides inheritance, macros, filters, and a set of Zola-specific built-in functions — `get_url`, `get_page`, `load_data`, `get_taxonomy`, `resize_image` — that reach back into the site graph and filesystem. Shortcodes are small Tera templates invoked from Markdown, which is the sanctioned extension point in lieu of a plugin API. Because there is no user-supplied compiled code, the whole render is deterministic and sandboxed to what the binary exposes.

The binary is self-contained: a `zola` build has no runtime dependencies and the same executable serves `init`, `serve` (with file-watching live reload), `build`, and `check` (an internal/external link checker).

## Production Notes

**No plugin system, by design.** This is the first thing to internalize. Custom build logic is not possible inside Zola; `load_data` (fetch local/remote JSON, TOML, CSV) and shortcodes cover a surprising amount, but anything beyond that means preprocessing content before Zola runs. Do not adopt Zola expecting to "write a plugin later."

**Still pre-1.0.** After years of use, Zola remains on the 0.x series, and minor releases have carried breaking changes to config, template variables, or Markdown handling. Pin the Zola version in CI (the release binary or `cargo install zola --version`) and read the CHANGELOG before bumping — a silent template-variable rename can break a build that worked yesterday.

**License is split.** Code introduced after version 0.22 is EUPL-1.2; code predating a specific 2024-era commit stays MIT[^3]. The EUPL is a copyleft license with an explicit compatibility list. If your organization vets licenses, note that Zola is not uniformly MIT anymore — this matters if you vendor or fork the source, though it does not affect sites you generate with the binary.

**Multilingual is "basic."** The README itself flags multilingual support as basic. It handles per-language content and sections, but teams with heavy i18n needs (complex fallbacks, per-locale taxonomies, translation workflows) frequently find it thinner than Hugo's.

**Theme ecosystem is small.** Zola's theme gallery is a fraction of Hugo's or Jekyll's. Expect to build or heavily adapt a theme rather than drop one in, and expect some community themes to lag behind breaking Zola releases.

**Build performance.** Fast for typical sites; the usual bottleneck at scale is image processing (`resize_image`) and syntax highlighting across very large content sets, not Markdown parsing. Builds are single-run and cache-light, so CI rebuilds everything each time.

## When to Use / When Not

**Use when:**
- You want a single binary with no Node/Ruby/Go toolchain and no plugin management.
- You want Sass, build-time syntax highlighting, client-side search, and image resizing without wiring them together.
- You prefer Jinja2/Tera-style templates over Go templates.
- The site is a blog, docs, or landing site that fits Zola's built-in feature set.

**Avoid when:**
- You need build-time extensibility or custom compiled logic — there is no plugin API.
- You depend on a large drop-in theme marketplace (Hugo, Jekyll).
- You need advanced, workflow-heavy multilingual support.
- You want component-driven interactivity or partial hydration (reach for a JS framework generator instead).

## Alternatives

- gohugoio/hugo — much larger theme ecosystem and feature surface with Go templates; use when you want maximum themes/features and don't mind Go's template language.
- 11ty/eleventy — JS-based, highly flexible data cascade; use when your team lives in the JS ecosystem and wants scripting freedom.
- jekyll/jekyll — Ruby, native to GitHub Pages; use for legacy sites or GitHub Pages defaults.
- getpelican/pelican — Python static generator; use when a Python toolchain is preferred.
- withastro/astro — component-based with interactive islands; use when you need React/Vue/Svelte components and partial hydration, not just static HTML.

## History

| Version | Date | Notes |
|---------|------|-------|
| Gutenberg 0.1 | 2017 | Initial release under the name Gutenberg[^1]. |
| 0.5 (renamed Zola) | 2018-11 | Renamed from Gutenberg to Zola to avoid the WordPress Gutenberg clash[^1]. |
| 0.22 | 2024 | New code relicensed to EUPL-1.2; pre-existing code stays MIT[^3]. |

## References

[^1]: getzola/zola README — "zola (né Gutenberg)". https://github.com/getzola/zola/blob/master/README.md
[^2]: getzola/zola README, feature list and design rationale; Zola documentation. https://www.getzola.org/documentation/getting-started/overview/
[^3]: getzola/zola README, License section — EUPL-1.2 for code after 0.22, MIT for prior code. https://github.com/getzola/zola/blob/master/LICENSE

## Tags

rust, static-site-generator, ssg, markdown, tera, jamstack, blog-engine, documentation-tool, cli, single-binary
