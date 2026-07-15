# marp-team/marp-core

> The rendering engine behind Marp — a Markdown-to-slide-deck converter that turns a `.md` file into HTML + CSS.

[GitHub repo](https://github.com/marp-team/marp-core) ·
[License: MIT](https://github.com/marp-team/marp-core/blob/main/LICENSE)

## Overview

Marp Core is the rendering library at the center of the **Marp** (Markdown Presentation) ecosystem. It takes Markdown text and returns an HTML string plus a CSS string that together form a slide deck. It is a library, not an application: the tools people actually run — Marp CLI, the Marp for VS Code extension, Marp Web — are thin wrappers that call `marp-core` to do the conversion and then handle files, preview, or PDF/PPTX export themselves[^1].

The project descends from an earlier Electron desktop app also called Marp. Around 2018 the team abandoned the monolithic app and re-architected it into three layers: **Marpit** (the framework that handles slide splitting, directives, and the CSS theming contract), **marp-core** (an opinionated Markdown flavor built on Marpit), and downstream tools[^2]. `marp-core` extends the `Marpit` class and adds the things a general-purpose slide tool needs but Marpit deliberately leaves out: built-in themes, emoji, math typesetting, and auto-scaling text.

The defining tension is this split. Marpit is intentionally minimal and unopinionated (no themes, no math, bring your own CSS); marp-core is the batteries-included layer with three official themes and sensible defaults. If you only ever use Marp CLI or the VS Code extension you are using marp-core without knowing it, and most of its configuration surface is reachable only through global directives in the Markdown front-matter or a JS constructor most users never touch.

## Getting Started

```bash
npm install --save @marp-team/marp-core
```

```javascript
import { Marp } from '@marp-team/marp-core'

const marp = new Marp()
const { html, css } = marp.render('# Hello, marp-core!')
// You assemble the final document yourself:
// `<style>${css}</style>` + `${html}`
```

`render()` returns the slide markup and the theme CSS separately; it does not emit a complete HTML document. Wiring them into a page (and injecting the browser helper script) is the caller's job — which is exactly what Marp CLI does for you.

## Architecture / How It Works

`marp-core` is a subclass of Marpit. Markdown parsing runs through **markdown-it**, the same parser Marpit uses, with marp-core registering additional plugins for its extra syntax. The pipeline: Markdown → markdown-it tokens → Marpit slide splitting and directive handling → marp-core's added rendering (emoji, math, fit/auto-scale markers) → `{ html, css }`.

What marp-core adds on top of Marpit:

- **Marp Markdown flavor** — CommonMark plus GitHub-Flavored tables and strikethrough, paragraph line-breaks rendered as `<br>`, and heading slugification (auto `id` attributes) on by default. Marpit's inline-SVG mode, CSS container queries, and loose YAML are enabled by default here.
- **Three official themes** — `default`, `gaia`, `uncover`, selected via `<!-- theme: gaia -->`. These ship with the `@auto-scaling` and `@size` metadata that the extended features key off.
- **Math typesetting** — `$...$` / `$$...$$` rendered by **MathJax** (default) or **KaTeX**, chosen with the `math` global directive or constructor option. MathJax is preferred for rendering fidelity; KaTeX is faster for decks with many formulas.
- **Emoji** — shortcodes (`:smile:`) and Unicode emoji converted to **twemoji** SVG images by default, fetched from the jsDelivr CDN unless you point `twemoji.base` at local assets.
- **Auto-scaling** — the "fitting header" (`# <!-- fit -->`) and auto-shrink for overflowing code/KaTeX blocks. This depends on Marpit's inline-SVG mode; setting `inlineSVG: false` silently disables it.
- **HTML allowlist** — unlike raw markdown-it, marp-core sanitizes HTML against a default allowlist of safe tags/attributes. `html: true` disables sanitization entirely; passing an object customizes the allowlist, including per-attribute sanitizer functions.

Rendered decks rely on an injected **browser helper script** (`@marp-team/marp-core/browser`) that runs a WebKit inline-SVG polyfill and finalizes auto-scaled elements at view time. By default this is inlined into the output; the `script` option can switch it to a CDN reference or turn it off for bundler-controlled setups.

## Production Notes

- **You are probably not the target consumer.** marp-core is an internal engine. Unless you are building your own presentation tool, use Marp CLI or the VS Code extension — they handle file I/O, preview, watch mode, and PDF/PPTX/PNG export that marp-core itself does nothing about.
- **CDN dependencies by default.** Both twemoji emoji images and (for KaTeX) web fonts load from jsDelivr at render/view time. Air-gapped or CSP-restricted environments must set `twemoji.base`, `katexFontPath`, and `script.source: 'inline'` to avoid broken emoji, missing math fonts, or a blocked helper script. The `script.nonce` option exists specifically for CSP setups.
- **Auto-scaling is horizontal only.** The fit/shrink logic scales content to the slide width; tall content can still overflow the bottom of a slide, and the docs say so explicitly. Disabling inline SVG (`inlineSVG: false`) turns off fitting headers and auto-shrink with no error.
- **Math library is a real choice, not a toggle.** MathJax and KaTeX produce different output and support different LaTeX subsets. A deck authored against one may render subtly differently (or fail on unsupported macros) under the other. The `math` global directive overrides the constructor, but if the constructor disabled math, the directive cannot re-enable it.
- **Slug collisions with Marpit's anchor.** marp-core's `slug` option assigns `id`s to headings; Marpit's separate `anchor` option assigns `id`s to slides. Leaving both on while rendering untrusted user Markdown can inject conflicting `id`s into your host page — the docs recommend setting both to `false` when embedding third-party content.
- **HTML defaults are conservative but overridable.** `html: true` is an easy way to render arbitrary markup and an easy way to open an XSS hole when the Markdown is user-supplied. Prefer the object allowlist form over the boolean.

## When to Use / When Not

**Use when:**
- You are building a tool or service that programmatically converts Markdown to slides and want the batteries-included Marp behavior (themes, math, emoji, fit) rather than assembling it on Marpit yourself.
- You need deterministic, self-hostable slide rendering embeddable in a larger app.
- You want the exact same rendering as Marp CLI / VS Code, driven from your own code.

**Avoid when:**
- You just want to make slides — install Marp CLI or the VS Code extension instead; they wrap this library and add everything a user needs.
- You want a minimal, theme-agnostic framework with no opinions — use Marpit directly and skip marp-core's added weight (twemoji, MathJax/KaTeX).
- You need WYSIWYG editing, animations, or speaker-notes tooling — Marp is Markdown-source-first and does none of that.

## Alternatives

- marp-team/marpit — the lower framework marp-core is built on; use it when you want slide/directive/theming primitives without themes, math, or emoji baggage.
- marp-team/marp-cli — use this instead of marp-core when you actually want to produce PDF/PPTX/HTML files from the command line.
- hakimel/reveal.js — use when you want a browser-native, JS-driven deck with animations and plugins rather than a Markdown-to-static-HTML pipeline.
- gnab/remark — use when you want in-browser Markdown slides from a single HTML file with no build step.
- slidevjs/slidev — use when you're a Vue/developer audience wanting Markdown decks with components, live coding, and richer theming.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2018-06 | Repo created during the rewrite from the Electron Marp app into Marpit + marp-core[^2]. |
| 1.0.0 | 2020-01-13 | First stable release. |
| 2.0.0 | 2021-04-24 | Major version; API/dependency modernization. |
| 3.0.0 | 2021-11-22 | Major version bump (built on newer Marpit). |
| 4.0.0 | 2024-09-09 | Latest major line; current default. |
| 4.3.1 | 2026-07-04 | Most recent release at time of writing. |

Actively maintained: ~1.1k stars, releases roughly every few months, latest push July 2026. Development is led by Yuki Hattori ([@yhatt](https://github.com/yhatt)) under the marp-team organization.

## References

[^1]: `@marp-team/marp-core` README — install, usage, and feature reference. https://github.com/marp-team/marp-core
[^2]: Marpit framework documentation (the base class marp-core extends). https://marpit.marp.app
[^3]: Marp official site and ecosystem overview. https://marp.app

## Tags

typescript, markdown, presentation, slides, static-site, marp, marpit, markdown-it, javascript, rendering-engine, developer-tools
