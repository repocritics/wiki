# jekyll/jekyll

> A Ruby static site generator that turns Markdown and Liquid templates into a plain directory of HTML — the engine behind GitHub Pages.

[GitHub repo](https://github.com/jekyll/jekyll) ·
[Official website](https://jekyllrb.com) ·
[License: MIT](https://github.com/jekyll/jekyll/blob/master/LICENSE)

## Overview

Jekyll is a blog-aware static site generator written in Ruby, created by Tom
Preston-Werner in 2008 and popularized by his "Blogging Like a Hacker" post[^1].
It reads a directory of Markdown files, front matter, and Liquid templates and
writes a complete static site to `_site/` — no database, no server-side runtime
at request time. Its defining moment was becoming the default engine for GitHub
Pages, which meant any GitHub repository could publish a site for free, and
which anchored Jekyll as the reference tool for developer blogs, project docs,
and documentation sites through the 2010s[^2].

The project's central tension in 2026 is age versus incumbency. Jekyll is
mature, stable, and enormously well-documented, but it is no longer the fastest
or most flexible option — Hugo builds far quicker, and JavaScript-based
generators integrate more naturally with modern component tooling. What keeps
Jekyll relevant is GitHub Pages integration, a decade of accumulated themes and
tutorials, and a genuinely simple mental model: files in, HTML out. It stays
actively maintained across a large fork and watcher base, but development is
deliberately conservative rather than expansive.

## Getting Started

```bash
gem install bundler jekyll
jekyll new my-site
cd my-site
bundle exec jekyll serve   # builds _site/ and serves at localhost:4000 with live reload
```

```markdown
---
layout: post
title: "Hello Jekyll"
date: 2026-07-15
tags: [notes]
---

Content in **Markdown**. Liquid works too:

{% for tag in page.tags %}#{{ tag }} {% endfor %}
```

The `---` fenced block at the top is YAML front matter; its absence tells Jekyll
to copy a file through untouched. Any file *with* front matter is rendered.

## Architecture / How It Works

Jekyll is a batch compiler, not a server. A build is a pipeline: read the site
into memory (posts, pages, collections, data files, static assets), run each
renderable file through the Liquid template engine, then through a Markdown
converter, wrap the result in nested layouts, and write everything to `_site/`.

Key pieces:

- **Liquid** — the template language, originally from Shopify. It is
  deliberately sandboxed: templates cannot execute arbitrary Ruby, which is what
  makes GitHub's server-side rendering of untrusted repos safe. Logic lives in
  tags (`{% %}`) and output in objects (`{{ }}`).
- **Markdown** — kramdown is the default converter[^3]. Markdown is applied
  *after* Liquid, so a document is Liquid-evaluated first, then converted.
- **Collections** — generalizations of the original `_posts` concept (added in
  2.0). Any `_name/` directory can become a queryable collection of documents,
  which is how docs sites model chapters, staff, products, etc.
- **Front matter defaults & `_config.yml`** — global configuration and
  per-path defaults, so you can assign `layout: post` to everything under
  `_posts/` without repeating it.
- **Plugins** — Ruby gems that hook the build via generators, converters, tags,
  and hooks. This is where Jekyll's extensibility lives, and also where its
  GitHub Pages restrictions bite (see below).

The coupling that matters most is Jekyll ↔ GitHub Pages ↔ Ruby toolchain. The
site's behavior is defined by a `Gemfile` locking Jekyll and plugin versions;
reproducing a build means reproducing that Ruby environment, native extensions
and all.

## Production Notes

**GitHub Pages runs an old, locked Jekyll.** The native Pages build pins a
specific Jekyll version (historically the 3.9.x line) and a small allowlist of
"safe" plugins[^4]. If you need a newer Jekyll, arbitrary plugins, or a custom
build step, you must bypass native Pages and build with **GitHub Actions**,
publishing `_site/` as an artifact. Most non-trivial Jekyll sites now do exactly
this. Assuming `bundle exec jekyll build` behaves identically on Pages is a
common early mistake.

**Build time scales with site size, and rebuilds are whole-site.** Jekyll
re-renders everything by default. Large sites (thousands of documents) can take
tens of seconds to minutes per build. `--incremental` exists but is explicitly
experimental and does not track cross-document dependencies reliably, so it can
serve stale output; many teams leave it off and accept full rebuilds. This is
the single most-cited reason people migrate to Hugo.

**Liquid vs. code samples.** Because Liquid runs before Markdown, any literal
`{{` or `{%` inside a code block — common in Vue, Angular, Handlebars, or Jekyll
tutorials — will be interpreted as template syntax and can break the build. The
fix is wrapping such content in `{% raw %}…{% endraw %}`, a footgun that surprises
nearly every documentation author at least once.

**Ruby environment friction.** Jekyll inherits Ruby's dependency-management
reality: version managers (rbenv/rvm), Bundler, and native-extension gems.
Onboarding a non-Ruby contributor, or building in CI on a fresh runner, often
fails first on toolchain setup rather than on Jekyll itself. Pinning Ruby and
gem versions in the repo is essential for reproducible builds.

**Upgrade pains.** The 3.x → 4.x transition changed the default Sass converter
and tightened some defaults; themes and plugins written for 3.x sometimes need
updates. Because GitHub Pages lags the release line, a site can work locally on
Jekyll 4 and fail on native Pages, forcing the Actions workflow above.

## When to Use / When Not

**Use when:**
- You want a free, zero-config publishing path on GitHub Pages for a blog,
  project site, or docs.
- Your site is small-to-medium and content-first; build speed is a non-issue.
- You value a stable, heavily-documented tool with a huge theme ecosystem.
- Your team is comfortable with Ruby, or willing to treat the toolchain as a
  black box behind a Gemfile.

**Avoid when:**
- Build performance matters at scale (thousands of pages) — Hugo is far faster.
- You want tight integration with a JavaScript component framework or a modern
  asset pipeline — Astro, Eleventy, or a React framework fit better.
- You need dynamic/interactive rendering; Jekyll is static-only by design.
- Your contributors won't touch Ruby and you want to avoid that toolchain
  entirely.

## Alternatives

- gohugoio/hugo — single Go binary, dramatically faster builds, no runtime
  dependency; use when build speed or a dependency-free toolchain is the priority.
- 11ty/eleventy — JavaScript generator with a similar "files in, HTML out" model
  and multiple template languages; use when your stack is Node.js.
- withastro/astro — component-oriented, islands architecture, first-class MDX;
  use when you want interactive components alongside static content.
- getzola/zola — Rust single-binary generator with built-in Sass and search;
  use when you want Hugo-like speed with a simpler feature set.
- middleman/middleman — the other mature Ruby generator; use when you want Ruby
  but a more app-like asset pipeline than Jekyll offers.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2008-11 | Created by Tom Preston-Werner; "Blogging Like a Hacker"[^1]. |
| Pages launch | 2008-12 | Adopted as the GitHub Pages build engine[^2]. |
| 1.0 | 2013-05 | First stable release; drafts, new CLI. |
| 2.0 | 2014-05 | Collections, Sass/CoffeeScript support. |
| 3.0 | 2015-10 | Liquid profiler, experimental incremental regeneration. |
| 4.0 | 2019-08 | Faster builds, updated Sass converter, Ruby version bump[^5]. |

## References

[^1]: Tom Preston-Werner, "Blogging Like a Hacker" — 2008-11-17. https://tom.preston-werner.com/2008/11/17/blogging-like-a-hacker.html
[^2]: GitHub Pages documentation, "About GitHub Pages and Jekyll." https://docs.github.com/en/pages/setting-up-a-github-pages-site-with-jekyll/about-github-pages-and-jekyll
[^3]: Jekyll docs, "Configuration — Markdown." https://jekyllrb.com/docs/configuration/markdown/
[^4]: GitHub Pages, "Dependency versions" (supported Jekyll version and plugin allowlist). https://pages.github.com/versions/
[^5]: Jekyll blog, "Jekyll 4.0.0 Released" — 2019-08-20. https://jekyllrb.com/news/2019/08/20/jekyll-4-0-0-released/

## Tags

ruby, static-site-generator, jekyll, liquid, markdown, blog-engine, github-pages, kramdown, documentation, jamstack
