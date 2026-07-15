# uikit/uikit

> A component-rich CSS/JS front-end framework, rewritten in v3 to drop jQuery — developed and steered by YOOtheme.

[GitHub repo](https://github.com/uikit/uikit) ·
[Official website](https://getuikit.com) ·
[License: MIT](https://github.com/uikit/uikit/blob/develop/LICENSE.md)

## Overview

UIkit is a traditional "batteries-included" front-end framework: a large CSS
layer (grid, typography, forms, cards, navs) plus a JavaScript layer of
interactive components (modal, dropdown, slider, offcanvas, lightbox) that
initialize themselves from HTML attributes. It has been developed since 2013 by
YOOtheme GmbH, a German theme vendor, primarily as the design system behind
their commercial page builder (YOOtheme Pro for WordPress and Joomla)[^1]. That
origin matters: UIkit is open source and MIT-licensed, but its roadmap is set by
one company's product needs rather than a broad foundation.

The project has two eras. UIkit 2 was jQuery-based and used LESS. UIkit 3, whose
stable `3.0.0` was tagged in January 2019 after a long release-candidate period
in 2018[^2], is a from-scratch rewrite with no jQuery dependency, a vanilla-JS
component runtime, and both LESS and SASS sources. The v2 → v3 gap was a hard
break with essentially no automatic migration path.

The defining tension is popularity versus polish. UIkit is smaller and less
hyped than Bootstrap or Tailwind, but its component set is unusually complete and
visually coherent out of the box, and the attribute-driven JS means you can build
interactive UI with almost no JavaScript. The cost is a single-vendor governance
model and an architecture that fits server-rendered pages far better than modern
SPA frameworks.

## Getting Started

```bash
npm install uikit
# or load from a CDN — no build step required
```

```html
<!-- CSS + core JS + the SVG icon bundle (icons ship separately) -->
<link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/uikit@3/dist/css/uikit.min.css" />
<script src="https://cdn.jsdelivr.net/npm/uikit@3/dist/js/uikit.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/uikit@3/dist/js/uikit-icons.min.js"></script>

<!-- Components are driven by uk-* attributes; no init call needed -->
<div uk-grid>
  <div class="uk-width-1-2@m">Left</div>
  <div class="uk-width-1-2@m">Right</div>
</div>

<a href="#my-modal" uk-toggle>Open</a>
<div id="my-modal" uk-modal>
  <div class="uk-modal-dialog uk-modal-body">Hello</div>
</div>
```

```js
// Imperative API when you need it
UIkit.modal('#my-modal').show();
```

## Architecture / How It Works

The JS runtime is the interesting part. On load, UIkit registers a global
`MutationObserver` that scans the DOM for `uk-*` attributes (and `data-uk-*`
aliases) and instantiates the matching component against each element. Adding a
`uk-dropdown` attribute to a node that appears later — via AJAX, a template, or a
framework render — auto-initializes it without any explicit call. Each component
is defined through an internal reactive system (`UIkit.component()`) with
props, computed values, and lifecycle hooks, loosely Vue-like in shape but
bespoke and not exposed as a general framework.

The CSS is authored in **LESS as the canonical source**, with a SASS port
generated alongside it. Theming is done through preprocessor variables plus
"hooks" — empty mixins UIkit calls at defined points so you can inject rules
without editing framework files. Building a custom theme means compiling the
source yourself; the prebuilt `uikit.css` bundles every component.

Icons are a separate SVG system (`uikit-icons.js`) rather than an icon font, so
they must be loaded explicitly. The framework is modular in source — you can
import individual components in a custom build — but the distributed CDN/`dist`
files are monolithic.

The whole design assumes it owns the DOM. That is a clean model for
server-rendered or lightly-scripted pages, and an awkward one when a virtual-DOM
framework also wants to own the same nodes.

## Production Notes

**SPA integration is the recurring footgun.** Because components mutate the DOM
(cloning nodes, moving modals to `<body>`, wrapping elements), pairing UIkit with
React or Vue means fighting over who controls the tree. Route changes require
re-initializing components on freshly rendered markup, and UIkit's DOM rewrites
can desync a virtual DOM's expectations. There are community React/Vue wrappers,
but none are first-party and they tend to lag releases. UIkit is happiest in
multi-page apps, Joomla/WordPress themes, and server-rendered stacks.

**No tree-shaking in the prebuilt CSS.** Ship the full `uikit.css` and you carry
every component whether or not you use it. Trimming means adopting the LESS/SASS
source and a build pipeline — which is also the only supported way to theme
beyond a handful of exposed variables.

**LESS is canonical; SASS is generated.** If you theme in SASS, expect the SASS
variables and occasionally behavior to trail the LESS source. Mixing the two
customization paths is a known source of confusion.

**Single-vendor cadence.** Most commits come from YOOtheme staff, and priorities
track the commercial page builder. Releases are frequent (the `v3.25.x` line
ships small point releases regularly), but external PRs and long-tail issues can
sit — the open-issue count runs into the hundreds. SemVer is used, but minor
releases have occasionally adjusted component defaults.

**Icons and JS are opt-in loads.** Forgetting the separate `uikit-icons` bundle
(icons silently don't render) and loading order relative to your own scripts are
the two most common first-hour mistakes.

## When to Use / When Not

**Use when:**
- You're building server-rendered pages, a CMS theme (especially Joomla or
  WordPress via YOOtheme), or a prototype and want a complete, coherent component
  set without wiring JavaScript.
- You want interactivity (modals, sliders, off-canvas) from HTML attributes with
  minimal or no custom JS.
- You value a consistent built-in look over maximal design freedom.

**Avoid when:**
- Your app is a React/Vue/Svelte SPA that owns its own DOM — the runtime fights
  the virtual DOM and there is no first-party binding.
- You want utility-first styling with per-build tree-shaking (reach for Tailwind).
- You need the largest possible ecosystem, third-party template market, and
  community answers — Bootstrap and Tailwind are far bigger.
- You want headless, unstyled primitives to fully control appearance.

## Alternatives

- twbs/bootstrap — the incumbent full CSS+JS framework; use it when the biggest
  ecosystem, template market, and documentation depth matter most.
- tailwindlabs/tailwindcss — utility-first with build-time purging; use it when
  you want design control and minimal shipped CSS instead of prebuilt components.
- fomantic/Fomantic-UI — comparably component-rich, class-driven framework; use
  it when you like Semantic UI's naming and want a community-maintained fork.
- franken-ui/franken-ui — shadcn-style components built on UIkit + Tailwind
  tokens; use it when you want UIkit's component logic with a modern token system.
- foundation/foundation-sites — another traditional responsive framework; use it
  when you want a mature Bootstrap alternative with heavy SASS theming.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-07 | Repository created; jQuery + LESS era begins[^3]. |
| 2.27.5 | 2018-05-15 | Late UIkit 2 release; end of the jQuery line[^4]. |
| 3.0.0-rc.1 | 2018-05-15 | First v3 release candidate — jQuery removed, LESS+SASS. |
| 3.0.0 | 2019-01-14 | Stable v3: vanilla-JS runtime, attribute-driven components[^2]. |
| 3.1.0 | 2019-04-17 | First v3 minor. |
| 3.10.0 | 2022-01-12 | Ongoing component and theming additions. |
| 3.20.0 | 2024-04-23 | Continued v3.x line. |
| 3.25.20 | 2026-07-14 | Latest release at time of writing[^5]. |

## References

[^1]: UIkit README and site — "an Open Source project developed by YOOtheme." https://github.com/uikit/uikit and https://yootheme.com
[^2]: `v3.0.0` tag, dated 2019-01-14, via the GitHub tags API. https://github.com/uikit/uikit/releases/tag/v3.0.0
[^3]: Repository `created_at` 2013-07-18, via `gh api repos/uikit/uikit`.
[^4]: `v2.27.5` tag, dated 2018-05-15, via the GitHub tags API. https://github.com/uikit/uikit/tree/v2.27.5
[^5]: Latest release `v3.25.20`, published 2026-07-14, via `gh api repos/uikit/uikit/releases/latest`.

## Tags

css-framework, front-end, javascript, less, sass, ui-components, responsive-design, web-components, yootheme, no-build, mit-license
