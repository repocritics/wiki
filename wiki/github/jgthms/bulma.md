# jgthms/bulma

> A CSS-only framework built on Flexbox: components and modifiers via class names, no bundled JavaScript.

[GitHub repo](https://github.com/jgthms/bulma) ·
[Official website](https://bulma.io) ·
[License: MIT](https://github.com/jgthms/bulma/blob/main/LICENSE)

## Overview

Bulma is a CSS framework authored primarily by Jeremy Thomas (jgthms), first published in 2016[^1]. It ships as a single stylesheet: you write plain HTML, add semantic class names (`button`, `box`, `columns`), and apply state through modifier classes (`is-primary`, `is-large`, `is-active`). Unlike Bootstrap it bundles no JavaScript at all — interactive behavior (navbar toggles, modals, dropdowns) is left entirely to the consumer[^2]. This "style layer only" stance is the framework's defining decision.

The grid and layout system is built on Flexbox rather than floats or CSS Grid, which was novel in 2016 and made vertical centering and equal-height columns trivial at the time. Bulma's appeal has always been readability: the class vocabulary reads like English, there is no utility-class soup in the markup, and a designer can memorize most of it. The tradeoff is the same one every component framework carries — you inherit opinionated visual defaults and a fixed set of components, and stepping outside them means overriding CSS or recompiling Sass.

For most of its life Bulma sat at `0.x` and was customized only by recompiling its Sass variables. The 1.0 release (2024) was a substantial internal rewrite that moved theming onto native CSS custom properties and added first-class dark mode, changing how customization works without changing the class names in your markup[^3]. As of 2026 it remains widely used (~50k GitHub stars) but is maintained at a deliberate, low-frequency cadence by a very small team; long quiet periods between releases are normal, not a sign of abandonment.

## Getting Started

```sh
npm install bulma
# or: yarn add bulma
```

```css
/* import the prebuilt stylesheet */
@import "bulma/css/bulma.css";
```

Or drop in the CDN build with no toolchain at all:

```html
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bulma@1/css/bulma.min.css">

<section class="section">
  <div class="columns">
    <div class="column">
      <div class="box">
        <button class="button is-primary is-large">Save</button>
      </div>
    </div>
    <div class="column">
      <div class="notification is-warning">Heads up.</div>
    </div>
  </div>
</section>
```

There is no build step required for basic use — the shipped output is one CSS file[^2]. Customization beyond that means importing the Sass sources and overriding variables (see below).

## Architecture / How It Works

Bulma's source is authored in Sass (`.scss`), organized into layers: `utilities` (functions, mixins, derived variables), `base` (element resets and typography), `elements` (buttons, boxes, tags, titles), `components` (navbar, card, modal, dropdown, pagination), `grid` (columns and tiles), `helpers` (spacing, color, visibility), and `themes`. The published npm package includes both the compiled `css/bulma.css` and the full `sass/` tree so consumers can rebuild a trimmed version.

The single most important design fact is that **Bulma is a naming convention plus a stylesheet — nothing more.** A `class="button is-loading"` does not "do" anything; it just selects CSS rules. Any dynamic behavior (adding `is-active` to open a dropdown, toggling the mobile navbar) must be wired up by your own JavaScript or by a wrapper library. Many of the well-known Bulma integrations (Buefy for Vue, react-bulma-components, Fulma for F#) exist specifically to supply that missing interaction layer[^2].

Pre-1.0, theming worked by redefining Sass variables (`$primary`, `$family-sans-serif`, `$radius`) before importing Bulma and recompiling. Version 1.0 re-architected this around CSS custom properties: derived values (shades, hover states, dark-mode variants) are now emitted as `var(--bulma-*)` tokens, so a large amount of theming — including switching to dark mode — can happen at runtime without a Sass build[^3]. The class names in your HTML did not change across that boundary, which is why 1.0 was low-friction for existing markup even though it was a big internal change.

Browser compatibility historically relied on Autoprefixer to backfill Flexbox prefixes; Bulma targets current evergreen browsers and only partially supported old Internet Explorer[^1].

## Production Notes

- **You must bring your own JavaScript.** The most common surprise for newcomers is that the navbar burger, modals, dropdowns, and tabs are inert until you add JS to toggle their state classes. Budget for this, or adopt a framework-specific wrapper.
- **Full CSS is large; purge it.** Shipping the complete `bulma.css` sends styles for every component whether or not you use them. Production builds should run a purge/tree-shake pass (PurgeCSS, or importing only the Sass partials you need) to avoid a bloated stylesheet.
- **Specificity and overrides.** Because behavior lives entirely in class-selector CSS, overriding a component often means matching or exceeding Bulma's selector specificity. Deep customization can turn into a chase of `!important` or Sass-variable overrides; the 1.0 CSS-variable model relieves much but not all of this.
- **The 0.9 → 1.0 upgrade is a real migration.** Projects that customized heavily via Sass variables need to re-verify their overrides against the new CSS-variable theming, and some helper/variable names changed. It is not a drop-in for deeply themed codebases; read the migration notes before bumping[^3].
- **Maintenance cadence is slow by design.** Releases can be months or years apart (there was a multi-year gap before 1.0). If you need a framework with rapid iteration, frequent security-driven releases, or a large paid support ecosystem, factor that in. For a pure CSS framework this matters less than it would for runtime code.
- **Component set is fixed.** Bulma gives you what it gives you. There is no plugin architecture for adding new first-class components — you either compose from existing ones, hand-write CSS, or pull in a community extension.

## When to Use / When Not

**Use when:**
- You want readable, component-oriented class names and clean HTML rather than long utility strings.
- You are building a conventional dashboard, marketing site, or admin panel and Bulma's default components cover it.
- You want zero JavaScript coupling and will supply your own interaction logic (or use a framework wrapper).
- You want a Flexbox grid and sensible defaults without configuring a build pipeline.

**Avoid when:**
- You need fine-grained, design-token-driven control and minimal shipped CSS — a utility framework fits better.
- You need batteries-included interactive components (working modals, dropdowns) out of the box with no extra wiring.
- You depend on a fast-moving upstream with frequent releases and a large commercial support ecosystem.
- Your design diverges heavily from Bulma's look; you will spend more time overriding than you save.

## Alternatives

- tailwindlabs/tailwindcss — utility-first; use instead when you want composable low-level classes and purge-driven minimal CSS rather than prebuilt components.
- twbs/bootstrap — component framework that ships JavaScript; use instead when you need working interactive widgets out of the box.
- picocss/pico — minimal, largely classless; use instead when you want semantic HTML styled with almost no class names.
- daisyui/daisyui — component layer on top of Tailwind; use instead when you want Bulma-like named components but Tailwind's tooling underneath.
- foundation/foundation-sites — older component framework; use instead when you need its email/CSS-grid tooling or existing Foundation expertise.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2016 | Initial public release; Flexbox grid and core components[^1]. |
| 0.7.x | 2018 | Steady component growth while at 0.x. |
| 0.9.0 | 2020 | Late-0.x line; Sass-variable customization model. |
| 1.0.0 | 2024 | Rewrite onto CSS custom properties; native dark mode; theming without a Sass rebuild[^3]. |

(Pre-1.0 point releases were frequent but individually small; the 1.0 jump is the meaningful architectural boundary.)

## References

[^1]: Bulma README and project site — "a modern CSS framework based on Flexbox." https://bulma.io and https://github.com/jgthms/bulma
[^2]: Bulma README, "CSS only" — the sole output is a single CSS file and no JavaScript is included. https://github.com/jgthms/bulma#css-only
[^3]: Bulma 1.0 release and documentation — CSS variables and dark mode theming. https://bulma.io/documentation/ and https://github.com/jgthms/bulma/releases

## Tags

css, css-framework, flexbox, sass, frontend, ui-components, styling, design-system, no-javascript, responsive
