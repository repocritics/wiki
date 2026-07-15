# tabler/tabler-icons

> An MIT-licensed set of ~6,100 SVG icons drawn on a uniform 24×24 grid with a 2px stroke, distributed as raw files, a webfont, and framework component packages.

[GitHub repo](https://github.com/tabler/tabler-icons) ·
[Official website](https://tabler.io/icons) ·
[License: MIT](https://github.com/tabler/tabler-icons/blob/main/LICENSE)

## Overview

Tabler Icons is an open-source icon set started in 2020 by Paweł Kuna (codecalm) as a spin-off of the larger Tabler admin-dashboard UI project[^1]. It has grown into one of the larger free icon libraries — the repository advertises 6,146 icons as of mid-2026, split into roughly 5,093 outline and 1,053 filled variants[^2]. Every glyph is drawn to the same constraints: a 24×24 viewBox, a 2px nominal stroke, rounded line caps and joins. That consistency is the whole point — an icon set that visually matches across hundreds of glyphs is worth more than a larger set that doesn't.

The design language is close to Feather Icons (thin, rounded, stroked line art), and Tabler is often chosen as the "Feather but much bigger and still maintained" option. Because strokes are real SVG strokes rather than filled paths, the visual weight is a single CSS-adjustable property (`stroke-width`), which lets a downstream app tune icon weight to match its typography without swapping assets.

The defining tension is distribution surface versus bundle discipline. Tabler ships as raw SVGs, a sprite, a webfont, and per-framework component packages (React, Vue, Svelte, Angular via a third party). The component packages are convenient but each exports thousands of components, and naive imports can pull a great deal of code into a bundle if tree-shaking is misconfigured — the most common production complaint in the ecosystem.

## Getting Started

```bash
npm install @tabler/icons-react   # React
# or: @tabler/icons-vue, @tabler/icons-svelte, @tabler/icons (raw SVG + sprite)
```

```jsx
import { IconAward } from '@tabler/icons-react';

export default function Badge() {
  return (
    <IconAward
      size={36}          // sets width + height
      color="red"        // sets stroke color
      stroke={1.5}       // sets stroke-width
    />
  );
}
```

Raw SVG needs no build step at all — reference a file, inline the markup, or use the sprite:

```html
<svg width="24" height="24"><use xlink:href="tabler-sprite.svg#tabler-activity" /></svg>
```

## Architecture / How It Works

The source of truth is a directory of individual SVG files, one per icon per variant, authored against strict grid rules. Everything shipped is generated from those files:

- **`@tabler/icons`** — the raw SVG files plus a combined `tabler-sprite.svg` and per-icon nodes.
- **`@tabler/icons-webfont`** — a font compiled with FontForge from the SVGs, addressable via CSS classes (`<i class="ti ti-brand-tabler">`). The repo ships a `compile-options.json` mechanism to subset the font by icon name or category and to bake in a fixed stroke width, since a webfont cannot vary stroke at runtime.
- **`@tabler/icons-react` / `-vue` / `-svelte` / `-svelte-runes`** — codegen'd component wrappers. Each component is a thin function returning the SVG with props (`size`, `color`, `stroke`) mapped onto SVG attributes. Svelte 4 and Svelte 5 (runes) ship as separate packages because the component model changed between those versions.

Because the components are generated, they inherit the SVGs' uniformity for free, and a new icon added to the source directory propagates to every distribution channel on the next release. The React package exports its own TypeScript declarations, so `IconName` autocompletion and prop typing work without extra `@types` packages.

Angular support lives in a separate community-maintained package (`angular-tabler-icons` by pierreavn), not in this repo, which is worth noting for teams that expect first-party parity across frameworks.

## Production Notes

**Tree-shaking is the whole ballgame.** The framework packages export thousands of named components from a barrel. With a modern bundler (Vite, webpack 5, esbuild) and named imports, only the icons you use are included. But two footguns recur: (1) importing the whole namespace (`import * as Icons`) defeats shaking entirely and can add megabytes; (2) some setups — older Next.js, certain Jest/transform configs — resolve the barrel eagerly and either bloat the bundle or slow dev startup to a crawl. The usual fixes are per-icon deep imports where supported, `modularizeImports`/`optimizePackageImports`-style bundler hints, or preferring the raw SVG/sprite path for large icon counts.

**Dev-server cold start.** On large component barrels, dev tooling that pre-bundles dependencies can spend noticeable time processing thousands of tiny modules the first time. This is a one-time cost per dependency-optimization pass, not a runtime cost, but it surprises people.

**Webfont loses per-icon color/stroke control.** The font is a single baked stroke width; you pick it at compile time. If you need multiple weights you compile multiple fonts or use the SVG/component path instead. Fonts also render as text, so they inherit `color` but ignore `stroke-width` at runtime.

**Icon renames and removals between releases.** As the set is curated, some icons are renamed or deprecated across minor versions. Pinning a version and reviewing the changelog before bumping avoids "component is not exported" build breaks. The React package's type declarations turn most of these into compile-time errors rather than blank renders, which helps.

**Filled vs outline coverage is uneven.** Not every outline icon has a filled counterpart (roughly 1,053 filled against 5,093 outline). Designs that assume a filled variant exists for every glyph will hit gaps.

## When to Use / When Not

**Use when:**
- You want a large, visually consistent, permissively licensed icon set and value stroke-adjustable line art.
- You're on React, Vue, or Svelte and want typed components out of the box.
- You need multiple delivery formats (SVG, sprite, font) from one source.

**Avoid when:**
- You need filled/duotone/multicolor icons as the primary style — Tabler is stroke-first and filled coverage is partial.
- You want a tiny curated set and can't afford to configure tree-shaking carefully.
- You need brand/logo icons at scale — those exist but a dedicated brand-icon set may cover more.

## Alternatives

- feathericons/feather — the stylistic predecessor; smaller and less actively expanded, choose it when you want a minimal curated set.
- lucide-icons/lucide — a community fork of Feather with active development and strong framework packages; the closest direct competitor, prefer it if you want Feather lineage with momentum.
- twbs/icons — Bootstrap Icons, use when you're already in the Bootstrap ecosystem or want more filled variants.
- Ionicons (ionic-team/ionicons) — use when targeting Ionic/mobile with matching iOS/Material styles.
- phosphor-icons/core — use when you want a set with built-in multiple weights (thin/light/regular/bold/fill/duotone) as a first-class feature.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2020-02 | Split out of the Tabler dashboard project; outline SVGs only[^1]. |
| 1.x | 2020–2022 | Framework packages (React, Vue) and webfont added; set grows past 1,000 icons. |
| 2.x | 2023 | Filled variants introduced; package restructuring across `@tabler/icons-*`. |
| 3.x | 2024 | Continued expansion; Svelte 5 runes package split from Svelte 4. |
| — | 2026-07 | ~6,146 icons (≈5,093 outline / ≈1,053 filled); actively maintained[^2]. |

## References

[^1]: Tabler project (dashboard UI kit) and Tabler Icons origin. https://tabler.io/ and https://github.com/tabler/tabler
[^2]: Tabler Icons README, icon counts (6,146 total / 5,093 outline / 1,053 filled). https://github.com/tabler/tabler-icons

## Tags

javascript, svg, icons, icon-pack, react, vue, svelte, webfont, ui, open-source, mit-license, design-system
