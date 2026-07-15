# argyleink/open-props

> A library of CSS custom properties — design tokens for colors, sizes, shadows, easings, and more — shipped as plain CSS variables, not a framework.

[GitHub repo](https://github.com/argyleink/open-props) ·
[Official website](https://open-props.style) ·
[License: MIT](https://github.com/argyleink/open-props/blob/main/LICENSE)

## Overview

Open Props is a set of predefined CSS custom properties, authored by Adam Argyle,
first published in 2021[^1]. It is not a CSS framework in the Bootstrap/Tailwind
sense: it ships no components, no utility classes, and no layout system. What it
ships is a large flat namespace of `--` variables — `--size-3`, `--gray-6`,
`--font-size-4`, `--shadow-2`, `--ease-out-3`, `--radius-2`, `--gradient-7` — that
you reference from your own hand-written CSS. Argyle's framing is "sub-atomic
styles": the smallest reusable design decisions, decoupled from any opinion about
how you compose them[^2].

The defining tradeoff is scope. Because Open Props is just variables, it drops
cleanly into any stack (vanilla CSS, Sass, CSS Modules, styled-components,
Tailwind's `theme`) and imposes almost no lock-in — you can delete it and keep
your CSS. The cost is that it does nothing for you automatically: you still write
every selector, every rule, and every responsive breakpoint by hand. It replaces
the "magic numbers" in a design system, not the design system itself.

The second tension is delivery size. The full token set expands to thousands of
variables. Importing everything is convenient but ships far more CSS than a typical
project uses, which is why the project pushes a just-in-time (JIT) build path for
production (see Production Notes).

## Getting Started

```bash
npm install open-props
```

```css
/* Import everything (simplest; largest) */
@import "open-props/style";
@import "open-props/normalize";   /* optional CSS reset, light + dark */

.card {
  padding: var(--size-3);
  border-radius: var(--radius-2);
  box-shadow: var(--shadow-3);
  background: var(--gray-0);
  color: var(--gray-9);
  font-size: var(--font-size-2);
  transition: transform var(--ease-out-3) 200ms;
}
```

Or via CDN with no build step at all:

```html
<link rel="stylesheet" href="https://unpkg.com/open-props" />
<link rel="stylesheet" href="https://unpkg.com/open-props/normalize.min.css" />
```

## Architecture / How It Works

Open Props is generated, not hand-maintained. The source of truth is a set of
JavaScript modules under `src/` that describe each prop; a build step
(`npm run gen:op`) emits the PostCSS/CSS files, and `npm run bundle` produces the
minified distributables[^3]. Consumers never touch the JS — they import compiled
CSS — but the same JS modules are what make the JIT path possible, because they can
be read programmatically to know which props exist.

Every variable is declared on a low-specificity `:where()` selector rather than a
plain `:root`. This is deliberate: `:where()` contributes zero specificity, so the
tokens never fight your own selectors and are trivial to override or theme. Because
some environments (shadow DOM, older tooling, contexts where `:where()` is
undesirable) need different scoping, the repo ships variant builds via dedicated
generators: `gen:nowhere` (plain selectors), `gen:shadowdom` (`:host` scoping), and
`gen:prefixed` (every prop renamed `--op-*` to avoid collisions)[^3].

Tokens are also exported in non-CSS formats — a design-tokens JSON, a Figma tokens
sync file, and a Style Dictionary file — so the same values can drive design tools
and cross-platform token pipelines, not just the browser.

A few categories behave differently from the rest. The media-query props (e.g.
`--md`, `--lg`, adaptive light/dark hooks) rely on CSS `@custom-media`, which is not
natively supported by browsers; they only resolve if you run PostCSS with a
custom-media plugin. The rest of the props are ordinary CSS variables that work with
no build tooling.

## Production Notes

- **Ship the JIT build, not the whole thing.** The full stylesheet is convenient but
  large relative to what any one project uses. The recommended production path is
  `postcss-jit-props`, which scans your CSS and injects only the props you actually
  reference[^4]. Wire it into `postcss.config.js` by passing the `open-props` module:

  ```js
  const OpenProps = require('open-props');
  const jit = require('postcss-jit-props');
  module.exports = { plugins: [ jit(OpenProps) ] };
  ```

- **Custom-media props are a build-time feature.** `@custom-media`-based breakpoints
  will silently do nothing if you import the raw CSS without a PostCSS custom-media
  plugin. Teams on a no-build/CDN setup should treat the media-query props as
  unavailable and write plain `@media` queries.

- **The color scales are static, not a theming engine.** Open Props gives you fixed
  palettes (`--red-0`…`--red-12`, HSL variants, gradients). It does not generate a
  themeable token system with semantic aliases; you build that layer yourself by
  mapping semantic names (`--surface`, `--text`) onto Open Props values. The
  `normalize.css` handles light/dark at the reset level, but app-level theming is
  your responsibility.

- **Prop names are a stable public API you re-export.** Because you sprinkle
  `var(--size-3)` throughout your codebase, you inherit Open Props' naming as your
  own contract. Switching off it later is a find-and-replace across every stylesheet.
  Many teams alias the props behind their own semantic layer to insulate against this.

- **Version pinning matters at the CDN.** `unpkg.com/open-props` (unversioned)
  follows the latest release; pin a version for reproducible builds if the scale
  values matter to your design.

## When to Use / When Not

**Use when:**
- You write your own CSS and want a curated, consistent set of values instead of
  inventing magic numbers.
- You want design tokens with near-zero lock-in that work across vanilla CSS, Sass,
  CSS-in-JS, or as inputs to Tailwind/Style Dictionary.
- You need the same tokens in Figma and in code from one source.

**Avoid when:**
- You want a utility-class workflow (`class="p-3 shadow-md"`) — that is Tailwind's
  model, not this one.
- You want prebuilt components or layout primitives; Open Props ships none.
- You need a fully semantic, runtime-themeable token system out of the box and don't
  want to build the aliasing layer yourself.

## Alternatives

- tailwindlabs/tailwindcss — use when you want a utility-class methodology with
  content-based purging, rather than raw variables you apply by hand.
- yeun/open-color — use when you only need an accessible color palette exposed as
  variables, without sizes/shadows/easings.
- radix-ui/colors — use when you need perceptually-uniform color scales with built-in
  dark mode and alpha variants.
- system-ui/theme-specification (Theme UI / styled-system lineage) — use when your
  tokens should live in JS/React theme objects instead of CSS.
- picocss/pico — use when you want a classless base stylesheet you can drop in, not a
  token library to compose.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial release | 2021-09 | Repo created; "sub-atomic styles" CSS-variable approach by Adam Argyle[^1]. |
| 1.x stable | 2022 | Reached 1.0; JIT via `postcss-jit-props`, JSON/Figma/Style-Dictionary token exports[^4]. |
| Latest push | 2026-01-31 | Ongoing maintenance; variant builds (`nowhere`, `shadowdom`, `prefixed`) generated from `src/`. |

## References

[^1]: Open Props repository and homepage. https://open-props.style
[^2]: Adam Argyle, Open Props documentation — "sub-atomic styles" framing. https://open-props.style
[^3]: Open Props README — generator scripts (`gen:op`, `gen:nowhere`, `gen:shadowdom`, `gen:prefixed`, `bundle`). https://github.com/argyleink/open-props#cli
[^4]: `postcss-jit-props` — just-in-time injection of referenced props. https://github.com/GoogleChromeLabs/postcss-jit-props

## Tags

css, design-tokens, css-custom-properties, css-variables, design-system, frontend, styling, postcss, theming, adam-argyle
