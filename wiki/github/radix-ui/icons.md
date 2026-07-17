# radix-ui/icons

> A small, curated set of 15×15 icons shipped as tree-shakeable React components.

[GitHub repo](https://github.com/radix-ui/icons) ·
[Official website](https://radix-ui.com/icons) ·
[License: MIT](https://github.com/radix-ui/icons/blob/main/LICENSE)

## Overview

Radix Icons is the icon set that accompanies the Radix UI ecosystem (Radix
Primitives, Radix Colors, Radix Themes). Every glyph is drawn on a fixed 15×15
pixel grid and distributed, via the npm package `@radix-ui/react-icons`, as an
individually importable React component. It was designed by the team originally
behind Modulz, which was acquired by WorkOS in 2022 — the package copyright now
reads "2022–present WorkOS"[^1].

The defining characteristic is deliberate smallness. This is a curated set of a
few hundred common interface icons — arrows, chevrons, toggles, common actions —
in a single visual weight, not a comprehensive library. There are no size
variants, no outline/filled pairs, and no icon-font or sprite build; the deliverable
is React components (plus raw SVGs available on the website and inside the package).
That constraint is the whole point: at 15px, in one weight, the icons are visually
uniform and pair cleanly with Radix Themes' UI. It is also the main limitation —
if your design calls for a glyph that isn't in the set, or a heavier stroke at 24px,
there is no built-in path to get it here.

The repository itself is low-churn. It is a design artifact more than an actively
evolving codebase: the icon inventory changes rarely, and requests for new icons
frequently sit open for long stretches. As of this writing it carries ~2.7k stars
and 138 forks — a widely used staple within the Radix world rather than a
standalone icon powerhouse like Lucide or Tabler.

## Getting Started

```bash
npm install @radix-ui/react-icons
```

```tsx
import { FaceIcon, ImageIcon, SunIcon } from "@radix-ui/react-icons";

function Toolbar() {
  return (
    <div style={{ color: "var(--gray-11)" }}>
      {/* default renders at 15×15, fill="currentColor" */}
      <SunIcon />
      {/* size via width/height props */}
      <FaceIcon width={20} height={20} />
      {/* color via the `color` prop or inherited `currentColor` */}
      <ImageIcon color="tomato" />
    </div>
  );
}
```

Each icon is a named export. Color comes from `currentColor`, so the idiomatic
way to theme icons is to set CSS `color` on an ancestor rather than passing a
`color` prop per icon.

## Architecture / How It Works

The source of truth is a folder of hand-authored SVGs on a 15×15 grid. A build
step converts each SVG into a React component that renders inline `<svg>` with
`viewBox="0 0 15 15"`, `width={15} height={15}` defaults, and `fill="currentColor"`.
Icons spread incoming props onto the root `<svg>`, so anything valid on an SVG
element (`className`, `style`, `onClick`, `aria-*`, `width`/`height`) passes
through. There is a `color` convenience prop that maps to the fill.

Because every icon is a separate ESM named export, bundlers that perform
tree-shaking include only the icons you actually import. This is the key
distribution decision: you `import { CheckIcon }` and pay for one glyph, not the
whole set. The corollary is that the tree-shaking has to work — a CJS build path,
a bundler misconfiguration, or a barrel re-export that defeats dead-code
elimination can silently pull in far more than intended.

Naming is consistent: PascalCase with an `Icon` suffix (`ArrowRightIcon`,
`MagnifyingGlassIcon`, `Cross2Icon`). Numeric suffixes distinguish variants of a
concept (e.g. multiple cross/close weights). Beyond React, the raw SVGs are
downloadable from the website and are present in the published package, so
non-React consumers can extract them, but there is no first-party Vue/Svelte/web-
component wrapper.

## Production Notes

- **Designed for one size.** The glyphs are optimized for ~15px. Scaled up to
  24–48px they read as thin and under-detailed because there is no heavier
  weight to switch to. If a design system needs large iconography, this set is
  the wrong tool; use it for dense UI (toolbars, menus, form controls, tables).
- **No growth guarantee.** The inventory is effectively curated and near-frozen.
  Open issues and PRs proposing new icons can linger. Don't adopt it expecting a
  specific missing glyph to arrive on your timeline — plan to supplement with
  your own SVGs (which won't perfectly match the grid/weight) if you need more.
- **Tree-shaking is load-bearing.** Verify your production bundle actually drops
  unused icons. Prefer named imports; avoid re-exporting the whole package
  through an index barrel that a bundler can't shake.
- **Accessibility is on you.** Icons render as bare `<svg>` with no default
  `aria-label` or `role`. Add `aria-hidden` for decorative icons, or a label /
  `<title>` for meaningful ones.
- **Coupling is loose.** The package has no hard dependency on Radix Primitives
  or Themes — you can use the icons in any React app. It does assume React
  (`react` as a peer dependency) and JSX.

## When to Use / When Not

**Use when:**
- You're building with Radix Themes/Primitives and want visually matched icons.
- Your UI is dense and lives around 15px: dashboards, editors, toolbars, menus.
- You want React components with `currentColor` theming and real tree-shaking.
- You value a small, consistent, single-weight set over breadth.

**Avoid when:**
- You need large or marketing-scale iconography, or multiple stroke weights.
- You need a big, actively expanding library with a request-and-receive cadence.
- You're not on React and don't want to hand-manage extracted SVGs.
- Your brand needs an icon not in the set and consistency across the set matters.

## Alternatives

- lucide-icons/lucide — much larger, actively maintained, multiple sizes/weights and first-party packages for many frameworks; use when you need breadth and momentum.
- tabler/tabler-icons — very large free set with outline and filled variants; use when you need coverage across many concepts.
- feathericons/feather — minimal outline set with a similar restrained aesthetic; use when you want stroke-based minimalism (note lower maintenance activity).
- primer/octicons — GitHub's icon set at 16/24px with real size variants; use when you want two purpose-built sizes rather than one.
- react-icons/react-icons — aggregator that bundles many sets (including Radix) behind one API; use when you want to mix sets without multiple installs.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2019-05-02 | Repository opened under what became the Radix org[^2]. |
| WorkOS acquisition | 2022 | Modulz (Radix's originator) acquired by WorkOS; package copyright becomes "2022–present WorkOS"[^1]. |
| `@radix-ui/react-icons` | 2022 | Distributed as the icon package for the Radix ecosystem; adopted by Radix Themes. |
| last push | 2026-04-02 | Low-churn maintenance; icon inventory largely stable[^2]. |

## References

[^1]: Radix Icons README — authorship and "MIT License, Copyright © 2022-present WorkOS." https://github.com/radix-ui/icons/blob/main/README.md
[^2]: GitHub API metadata for radix-ui/icons (created 2019-05-02, last push 2026-04-02, MIT, ~2.7k stars, 138 forks). https://github.com/radix-ui/icons
[^3]: Radix Icons documentation and downloadable SVG/Figma assets. https://radix-ui.com/icons

## Tags

react, icons, svg, design-system, typescript, ui-components, iconset, tree-shaking, radix, frontend
