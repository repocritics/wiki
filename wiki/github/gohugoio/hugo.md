# gohugoio/hugo

> A single-binary static site generator in Go, built for build speed on content-heavy sites.

[GitHub repo](https://github.com/gohugoio/hugo) ·
[Official website](https://gohugo.io) ·
[License: Apache-2.0](https://github.com/gohugoio/hugo/blob/master/LICENSE)

## Overview

Hugo is a static site generator (SSG): it takes Markdown content plus Go
templates and emits a directory of static HTML/CSS/JS you deploy to any host or
CDN. It was created by Steve Francia (spf13) in 2013 and has been led since by
Bjørn Erik Pedersen (bep), who authors most of the engine[^1]. It compiles to a
single dependency-free binary — no Node, Ruby, or Python runtime is required for
the core workflow, which is the main reason it displaced Jekyll for many
docs/blog/marketing sites.

The defining characteristic is build speed. Hugo renders thousands of pages per
second and full-site builds for typical blogs and docs sites complete in
hundreds of milliseconds to a few seconds. That speed comes from Go's
concurrency and an all-in-memory build model, and it is the feature the project
optimizes hardest for. Note the repo is still on 0.x versioning after more than
a decade — Hugo has never declared 1.0, and minor releases (`0.x.0`) routinely
carry breaking changes[^2], which is the single biggest source of operational
surprise (see Production Notes).

The tradeoff for the single-binary speed model is the templating layer: Hugo
uses Go's `html/template` syntax, which is verbose, has idiosyncratic
whitespace and error handling, and is the most common ergonomic complaint. There
is no runtime plugin API — you extend Hugo through templates, shortcodes, and
Hugo Modules, not by loading arbitrary Go code.

## Getting Started

Install via package manager (Homebrew shown) or a prebuilt binary from releases:

```bash
brew install hugo          # macOS/Linux
hugo version               # confirm; note "extended" in the string if present
```

Create and run a site:

```bash
hugo new site my-site
cd my-site
git clone https://github.com/theNewDynamic/gohugo-theme-ananke themes/ananke
echo "theme = 'ananke'" >> hugo.toml
hugo new content posts/hello.md
hugo server -D            # dev server with live reload; -D includes drafts
```

A content file is Markdown with front matter (TOML/YAML/JSON):

```markdown
+++
title = "Hello"
date = 2026-01-01
draft = false
+++

Body content in **Markdown**, parsed by Goldmark.
```

Build the deployable site into `public/` with `hugo`.

## Architecture / How It Works

Hugo builds the entire site in a single in-memory pass:

1. **Content assembly** — files under `content/` are read; each becomes a `Page`
   with front-matter metadata. Directory structure maps to URL structure and to
   Hugo's section/taxonomy model.
2. **Markdown rendering** — the default parser is **Goldmark**, a
   CommonMark-compliant parser that replaced Blackfriday as the default around
   v0.60 (2019)[^3]. Render hooks let templates override how links, images,
   headings, and code blocks are emitted.
3. **Template execution** — Go's `html/template` renders pages. Template
   selection follows a "lookup order" by page kind, type, and layout — powerful
   but implicit, and a frequent source of "why is this template not applied"
   confusion.
4. **Asset pipeline (Hugo Pipes)** — `resources.*` functions handle Sass→CSS,
   PostCSS, JS bundling via esbuild, image processing (resize/crop/convert),
   minification, fingerprinting, and SRI hashing.

**Editions.** Hugo ships as `standard`, `deploy`, `extended`, and
`extended/deploy`[^4]. The `extended` edition adds LibSass support (embedded, via
CGO). LibSass was deprecated in v0.153.0 and is slated for removal; the
recommended path is Dart Sass, which works with any edition but requires
installing the `dart-sass` binary separately. The `deploy` edition adds direct
upload to GCS/S3/Azure buckets. Confusing the editions is a common setup error —
Sass features silently fail on a `standard` binary.

**Hugo Modules** reuse Go's module system to share content, assets, themes,
data, and config across repositories via Git[^5]. This replaced the older
git-submodule theme workflow for projects that need composability.

Everything is co-designed around the in-memory, all-at-once build. There is no
incremental production build: `hugo` rebuilds the whole site each run. The dev
server (`hugo server`) mitigates this with fast partial rebuilds triggered by
file-watching, but CI builds pay full cost every time.

## Production Notes

**Pin the Hugo version.** Because breaking changes land in `0.x.0` minor
releases, an unpinned CI (Netlify, GitHub Actions, Cloudflare Pages) can break a
site when the runner picks up a newer Hugo. Always set an explicit version
(`HUGO_VERSION` on Netlify, an exact version in your workflow). This is the most
common real-world Hugo incident.

**Edition mismatch.** `hugo version` must contain `extended` for Sass/SCSS via
LibSass and for some image codecs on older versions. Package repos and Docker
images vary in which edition they ship; a build that works locally can fail in CI
on `error: this feature is not available in your current Hugo version`.

**Dart Sass is a separate install.** With LibSass deprecated, Sass-heavy sites
must provision the `dart-sass` binary in CI in addition to Hugo. It is not
bundled.

**Memory and large sites.** Very large sites (tens of thousands of pages, heavy
`image` processing) can consume substantial RAM because the build model holds
state in memory; image resizing is the usual culprit. Cache the `resources/`
output directory in CI so processed images are not regenerated every build.

**Template ergonomics.** Go template error messages point at the template, not
your mental model; whitespace control (`{{-` / `-}}`), scoping with `$`, and the
context (`.`) dot are recurring stumbling blocks. Debugging leans on
`{{ printf "%#v" . }}`, `errorf`, and `hugo --printPathWarnings`.

**Theme coupling.** Themes are tightly bound to the Hugo version and template
lookup behavior they were written against; upgrading Hugo can silently change
output or break a theme's layouts. Vendor or pin third-party themes.

**No dynamic server.** The output is static. Anything requiring per-request
logic (auth, user-specific content, form handling) must be offloaded to external
services, serverless functions, or client-side JS.

## When to Use / When Not

**Use when:**
- You have a content-heavy site (docs, blog, marketing, portfolio) and want fast
  builds without a Node/Ruby toolchain.
- You want a single binary and a Git-based, Markdown-first authoring workflow.
- Build time on large sites matters and you don't need per-request rendering.
- You want first-class multilingual, taxonomy, and asset-pipeline support built in.

**Avoid when:**
- You need server-side rendering or dynamic, user-specific pages.
- Your team prefers component-based JS/JSX authoring (Astro, Eleventy with JSX,
  Next.js cover that better).
- You want a rich runtime plugin ecosystem — Hugo has none by design.
- Go templates are a non-starter for your authors and you can't absorb the
  learning curve.

## Alternatives

- getzola/zola — Rust single-binary SSG with a similar "no dependencies" pitch;
  use when you want Hugo's deployment simplicity with Tera templates instead of Go.
- 11ty/eleventy — JS-based SSG; use when you want flexible JS templating and
  npm-ecosystem integration and can accept slower builds.
- withastro/astro — component-islands framework; use when you want to author in
  React/Vue/Svelte components and ship minimal JS.
- jekyll/jekyll — Ruby SSG native to GitHub Pages; use when GitHub Pages'
  built-in build is a hard requirement.
- gatsbyjs/gatsby — React + GraphQL SSG; use when you need a React data layer and
  plugin ecosystem and don't mind heavier builds.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2013-07 | Initial release by spf13, written in Go[^1]. |
| 0.15 | 2015-11 | Multilingual and taxonomy groundwork era. |
| 0.32 | 2018-01 | Native image processing in the asset pipeline. |
| 0.56 | 2019-05 | Hugo Modules (Go-module-based composition)[^5]. |
| 0.60 | 2019-11 | Goldmark became the default Markdown parser[^3]. |
| 0.153 | 2026 | Embedded LibSass deprecated; Dart Sass recommended[^4]. |

## References

[^1]: Hugo — "About Hugo" and project history. https://gohugo.io/about/
[^2]: Hugo releases and changelogs (breaking changes noted per minor). https://github.com/gohugoio/hugo/releases
[^3]: Hugo docs — Goldmark Markdown configuration and render hooks. https://gohugo.io/getting-started/configuration-markup/
[^4]: gohugoio/hugo README — editions and LibSass deprecation notice (v0.153.0). https://github.com/gohugoio/hugo
[^5]: Hugo docs — Hugo Modules. https://gohugo.io/hugo-modules/

## Tags

go, static-site-generator, ssg, jamstack, markdown, cms, documentation, blog-engine, templating, single-binary
