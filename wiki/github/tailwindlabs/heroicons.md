# tailwindlabs/heroicons

> A curated set of MIT-licensed SVG UI icons from the Tailwind CSS team, shipped as raw SVGs and first-party React/Vue components.

[GitHub repo](https://github.com/tailwindlabs/heroicons) ·
[Official website](https://heroicons.com) ·
[License: MIT](https://github.com/tailwindlabs/heroicons/blob/master/LICENSE)

## Overview

Heroicons is a hand-drawn icon set by Steve Schoger and Adam Wathan, the same
team behind Tailwind CSS[^1]. It is deliberately small and opinionated: rather
than trying to cover every glyph a designer might want, it ships a consistent
set of general-purpose interface icons (navigation, actions, status, common
objects) drawn to a fixed grid. Because of the Tailwind lineage it is the
default icon set a large share of Tailwind projects reach for.

The distinguishing design decision is that each icon is authored at multiple
sizes rather than being one master path scaled up and down. Since v2 the set
ships four variants: 24×24 outline, 24×24 solid, 20×20 solid, and 16×16 solid[^2].
The smaller sizes are redrawn for their pixel budget — a 16px icon is not the
24px icon shrunk — which is why the set stays crisp in dense UI. Outline icons
use `stroke="currentColor"` with a 2px stroke; solid icons use `fill`.

The defining tension is scope. Heroicons is intentionally closed: the
maintainers explicitly do not accept new icons or new framework ports, and
direct users who want more to fork or build their own library[^3]. That keeps
the set coherent and its bundle small, but means teams that outgrow ~300 icons
must supplement it or switch to a larger, contribution-open set.

## Getting Started

Copy-paste from [heroicons.com](https://heroicons.com), or install a component
package:

```sh
npm install @heroicons/react   # or @heroicons/vue
```

```jsx
// Each icon is imported individually from a size/style subpath
import { BeakerIcon } from '@heroicons/react/24/solid'

function Flask() {
  // Components forward SVG props; size via CSS, not width/height attrs
  return <BeakerIcon className="size-6 text-blue-500" aria-hidden="true" />
}
```

Import paths encode the variant: `@heroicons/react/24/outline`,
`.../24/solid`, `.../20/solid`, `.../16/solid` (and the same under
`@heroicons/vue`). Names are UpperCamelCase and always suffixed `Icon`.

## Architecture / How It Works

The repository is not the distribution. Committed source is a directory of
optimized master SVGs organized by size and style. A build step wraps each SVG
into a framework component and emits the published npm packages
(`@heroicons/react`, `@heroicons/vue`); this is why the repo's primary language
registers as JavaScript despite the product being SVG art[^4].

Each icon compiles to its own ES module. There is no sprite sheet and no
runtime registry — the React/Vue component is a thin function that returns the
inline SVG and spreads incoming props onto the root `<svg>` element, forwarding
`ref`. Because every icon is a separate module with no shared barrel side
effects, tree-shaking bundlers include only the icons you actually import.

The SVGs themselves carry no `width`/`height` attributes; they rely on `viewBox`
plus whatever CSS you apply (Tailwind's `size-*` utilities, or plain
`width`/`height`). Color is inherited via `currentColor`, so `text-*` utilities
recolor them for free. This is what makes Heroicons feel native inside a
Tailwind codebase and slightly awkward outside one.

## Production Notes

- **Icons are inlined into your JS bundle, not loaded as assets.** Each imported
  icon becomes JavaScript that renders inline SVG. This is fine for tens of
  icons; if a page renders hundreds of icon instances, an SVG `<use>` sprite
  approach will produce less DOM and smaller markup than inlined components.
- **Import from the exact subpath.** `import { XIcon } from '@heroicons/react/24/solid'`
  tree-shakes cleanly. Avoid dynamic or namespace imports (`import * as`) that
  can defeat dead-code elimination and pull the whole style into the bundle.
- **No implicit size.** Because components omit `width`/`height`, an icon with
  no sizing class inherits the SVG default of 100% and can render enormous,
  filling its container. Always apply a size utility or explicit dimensions.
- **v1 → v2 is a breaking migration.** v2 redrew the icons, renamed many, and
  restructured import paths from `/outline` and `/solid` into the
  size-prefixed `/24/outline`, `/24/solid`, `/20/solid`, `/16/solid` layout[^2].
  There is no drop-in shim; migration is a find-and-replace of both names and
  paths.
- **Accessibility is on you.** Components render decorative SVGs with no
  `aria` defaults. Add `aria-hidden="true"` for decorative icons or
  `role="img"` + a label for meaningful ones.
- **The set will not grow to fit you.** With new-icon contributions closed,
  gaps are permanent from upstream. Plan to mix in a second icon source or
  maintain local additions rather than filing requests.

## When to Use / When Not

**Use when:**
- You are already on Tailwind CSS and want icons that inherit color and size
  from utility classes with zero config.
- You want a small, visually consistent set and value tree-shakeable per-icon
  imports in React or Vue.
- You need crisp icons at small sizes (16/20px) drawn for those pixel grids.

**Avoid when:**
- You need broad coverage (brand logos, niche domains, thousands of glyphs) —
  Heroicons is intentionally narrow.
- You want to contribute or request icons; the project does not accept them.
- Your framework is Svelte, Solid, Angular, etc. — there are no first-party
  ports, and the maintainers decline to add them.
- You render very large numbers of icons per page and want a sprite/font
  delivery model rather than inlined SVG components.

## Alternatives

- lucide-icons/lucide — large, contribution-open fork of Feather with ports for
  most frameworks; use when you need breadth and community icons.
- tabler/tabler-icons — several thousand MIT stroke icons; use when Heroicons is
  too small.
- feathericons/feather — the minimal stroke set Lucide descends from; use for a
  lighter, framework-agnostic base.
- radix-ui/icons — 15×15 icons tuned for design-system UI; use when building on
  Radix primitives.
- simple-icons/simple-icons — brand and logo marks; use for the coverage
  Heroicons deliberately excludes.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2020-02 | Repository created; outline + solid SVGs, v1 line[^4]. |
| 1.0 | 2020 | First tagged release: 24px outline and 20px solid. |
| 2.0 | 2022 | Full redraw; four variants (24 outline/solid, 20 solid, 16 solid); import paths restructured under size prefixes[^2]. |

## References

[^1]: Heroicons — "by the makers of Tailwind CSS." Project README and
    https://heroicons.com
[^2]: Tailwind Labs, "Heroicons v2.0" announcement (2022). Introduced the
    24/20/16 size variants and the size-prefixed import paths.
    https://tailwindcss.com/blog
[^3]: Heroicons README, "Contributing" — the maintainers accept bug fixes only
    and decline new icons or additional framework ports.
    https://github.com/tailwindlabs/heroicons#contributing
[^4]: GitHub repo metadata, tailwindlabs/heroicons (created 2020-02-24,
    MIT-licensed, primary language JavaScript from the component build step).
    https://github.com/tailwindlabs/heroicons

## Tags

icons, svg, react, vue, tailwindcss, ui-components, design-system, frontend, javascript, icon-library, mit-license
