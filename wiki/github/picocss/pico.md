# picocss/pico

> A classless-first CSS framework that styles semantic HTML directly, so a plain document looks designed with no markup changes.

[GitHub repo](https://github.com/picocss/pico) ·
[Official website](https://picocss.com) ·
[License: MIT](https://github.com/picocss/pico/blob/main/LICENSE.md)

## Overview

Pico is a small CSS framework whose central bet is that most of a page's
appearance should come from the HTML elements themselves — `<article>`,
`<nav>`, `<table>`, `<form>`, `<button>`, `<dialog>` — rather than from a
vocabulary of utility classes you sprinkle onto `<div>`s. In its default
"class-less" mode you link one stylesheet and write ordinary semantic markup;
the result is a coherent, responsive, light/dark-aware page with essentially
zero authored CSS. This puts it at the opposite end of the spectrum from
utility frameworks like Tailwind: Pico is closer to "a reset stylesheet on
steroids" than to a component toolkit[^1].

The framework is CSS/SCSS only — there is no JavaScript, no build step
required for the CDN path, and no runtime. Interactive components are built on
native HTML that already has behavior: accordions are `<details>`, modals are
`<dialog>`, dropdowns are `<details class="dropdown">`. Theming runs on CSS
custom properties (all prefixed `--pico-`) and a `data-theme` attribute plus
`prefers-color-scheme`. The defining tradeoff falls out of the semantic bet:
because Pico styles bare tags globally, dropping it into an app that already
has its own CSS causes broad conflicts. Pico's answer is the "conditional"
build, which scopes every rule under a `.pico` wrapper so the framework only
affects an opt-in subtree[^2].

Pico is a good fit for content sites, prototypes, docs, admin panels, and
small apps where you want something that looks intentional without a design
system. It is explicitly not a large-project solution on its own: it ships no
helper/utility classes, so anything beyond its built-in elements and handful
of components is left to you and your own SCSS or CSS[^3].

## Getting Started

```html
<!-- CDN, class-less default -->
<link rel="stylesheet"
  href="https://cdn.jsdelivr.net/npm/@picocss/pico@2/css/pico.min.css">
```

```shell
npm install @picocss/pico     # or: yarn add @picocss/pico
```

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <meta name="color-scheme" content="light dark">
    <link rel="stylesheet" href="css/pico.min.css">
    <title>Hello world!</title>
  </head>
  <body>
    <main class="container">
      <h1>Hello world!</h1>
      <button>A styled button, no classes needed</button>
    </main>
  </body>
</html>
```

For SCSS customization, import the source and override the `$` settings maps
before compiling: `@use "pico" with ($theme-color: "azure");`. Pico is also
distributed via Composer (`composer require picocss/pico`) for PHP projects.

## Architecture / How It Works

Pico ships as a set of precompiled bundles rather than one file. The main
variants are: the default (needs `.container` for width and expects `.grid`
etc. where relevant), the **class-less** build (`pico.classless.css`, where
`<header>`/`<main>`/`<footer>` under `<body>` become centered containers
automatically), the **fluid class-less** build (full-width containers), and
the **conditional** build (`pico.conditional.css`, all selectors nested under
`.pico`)[^2]. Each of these also has a minified twin and colored-theme
variants. The color system in v2 precompiles ~20 accent themes into over 100
CDN-served combinations[^4].

Under the hood everything is SCSS compiled to CSS with a layer of custom
properties on top. The SCSS side is a settings-driven module system: you turn
whole feature groups (components, forms, specific elements) on or off through
maps, which is how Pico keeps the shipped CSS small — you only compile the
parts you use. The CSS side exposes the `--pico-*` variables so that
non-SCSS users can still retheme at runtime by overriding properties on
`:root`, on `[data-theme]`, or on any subtree.

Responsiveness is built into the element styles themselves rather than exposed
as breakpoint utilities. Typography scales with viewport, the grid uses native
CSS Grid, and spacing derives from a small set of variables. There is
deliberately no `.mt-4`/`.text-center`-style utility layer; the project treats
the absence of utilities as a feature, not a gap[^3].

## Production Notes

- **Global scope is the main footgun.** The default and class-less builds
  restyle bare HTML elements site-wide. In any app that already carries its
  own styles (or a component library), this collides immediately. Use the
  `pico.conditional.*` build and wrap Pico's subtree in `.pico`, or you will
  spend your time fighting specificity[^2].
- **It intentionally has no utilities.** There is no escape hatch of atomic
  classes. Past a certain complexity you are writing your own SCSS/CSS on top,
  which is the documented intended workflow — Pico is a starting point, not a
  finish line[^3].
- **Interactive components inherit native quirks.** Because modal =
  `<dialog>`, accordion = `<details>`, and dropdown = `<details>`, you get
  browser-native accessibility and behavior for free, but also the native
  limitations (e.g. `<details>` toggle semantics, `<dialog>` focus handling).
  There is no JS layer to paper over cross-browser edge cases.
- **Browser support is modern-only.** Latest stable Chrome, Firefox, Edge, and
  Safari are supported; no IE, including IE 11[^5]. The reliance on CSS custom
  properties, `:has()`-adjacent selectors, and native elements makes old
  browsers a non-goal.
- **Customize in SCSS, not by overriding compiled CSS.** Retheming through
  `--pico-*` variables is supported and clean; forking the compiled selectors
  is brittle across minor versions since class internals can change.
- **Release cadence has slowed.** The last tagged release, v2.1.1, is from
  March 2025, though the repository has commits into 2026. For a dependency
  this stable that is usually fine — the surface is small — but do not expect
  frequent feature releases.

## When to Use / When Not

**Use when:**
- You want a semantic HTML document to look designed with near-zero authored
  CSS (docs, blogs, landing pages, prototypes, internal tools).
- You value native accessibility and light/dark support out of the box.
- You want a single small stylesheet with no JS and no bundler requirement.
- Your team prefers writing HTML over composing utility classes.

**Avoid when:**
- You are dropping styles into an app that already has its own CSS and cannot
  isolate a subtree (or won't use the conditional build).
- You need a broad prebuilt component catalog with JS behavior (carousels,
  complex date pickers, toasts) — Pico deliberately ships very little.
- You want utility-class ergonomics for rapid, class-driven layout.
- You need to support legacy browsers.

## Alternatives

- kognise/water.css — even more minimal classless drop-in; use when you want a single `<link>` with literally nothing to configure and no SCSS.
- kevquirk/simple.css — classless framework in the same niche with slightly different defaults; use when Pico's look doesn't fit but you still want zero classes.
- tailwindlabs/tailwindcss — utility-first, the philosophical opposite; use when you want to compose everything from atomic classes rather than semantic defaults.
- twbs/bootstrap — large class-based component library with JS; use when you need a big catalog of prebuilt, cross-browser components and grid classes.
- pure-css/pure — small modular class-based framework; use when you want opt-in modules with classes but a tiny footprint.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.4.1 | 2021-10-25 | Mature v1 line; classless-first framework, CSS-variable theming. |
| v1.5.0 | 2022-03-13 | Continued v1 refinements. |
| v2.0.0 | 2024-02-11 | Major rewrite: SASS customization overhaul, full color palette, group component, ~20 precompiled themes, conditional styling[^4]. |
| v2.0.6 | 2024-03-03 | v2 bug-fix train. |
| v2.1.0 | 2025-03-15 | Latest minor line. |
| v2.1.1 | 2025-03-15 | Most recent tagged release. |

## References

[^1]: Pico CSS README — "A Superpowered HTML Reset". https://github.com/picocss/pico
[^2]: Pico CSS docs, "Conditional styling". https://picocss.com/docs/conditional
[^3]: Pico CSS docs, "Usage scenarios" / Limitations. https://picocss.com/docs/usage-scenarios
[^4]: Pico CSS, "What's new in v2". https://picocss.com/docs/v2
[^5]: Pico CSS README — "Browser Support". https://github.com/picocss/pico

## Tags

css, css-framework, scss, classless-css, semantic-html, dark-mode, minimal, frontend, styling, no-javascript
