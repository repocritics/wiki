# lucide-icons/lucide

> A community-maintained fork of Feather Icons that grew into a 1600+ icon set shipped as tree-shakeable components for every major frontend framework.

[GitHub repo](https://github.com/lucide-icons/lucide) ·
[Official website](https://lucide.dev) ·
[License: ISC](https://github.com/lucide-icons/lucide/blob/main/LICENSE)

## Overview

Lucide is an open-source icon library. It began in 2020 as a fork of Cole Bemis's Feather Icons, which had stalled with a small fixed set and little maintenance activity[^1]. The fork's premise was social rather than technical: keep the same 24×24, 2px-stroke aesthetic, but accept community-contributed icons instead of gatekeeping to one author. That decision is why the set has grown from Feather's ~280 icons to 1600+[^2] while keeping a recognizable visual language.

The project is really two things bolted together. One is a curated collection of SVG source files governed by strict design rules. The other is a build pipeline that turns those SVGs into idiomatic components for React, Vue, Svelte, Solid, Preact, Angular, Astro, React Native, and vanilla JS/DOM, plus a `lucide-static` package of raw files[^3]. Most users never touch the SVGs; they install one of the framework packages and import icons by name.

The defining tension is breadth versus consistency. Community contribution is what makes the set large and current, but every accepted icon is a permanent maintenance and visual-consistency liability. Lucide manages this with an explicit design spec and a hard editorial line — most notably a standing refusal to accept brand logos, on legal, consistency, and maintenance grounds[^4]. As of 2026 the repo has ~23.4k stars and is pushed to daily; it is one of the most actively maintained icon projects in the ecosystem.

## Getting Started

```bash
npm install lucide-react        # React
# or: lucide, @lucide/vue, @lucide/svelte, lucide-solid,
#     lucide-preact, @lucide/angular, @lucide/astro, lucide-react-native
```

```tsx
// React — each icon is its own module, so unused icons tree-shake away
import { Camera, ArrowRight } from "lucide-react";

export function Toolbar() {
  return (
    <div>
      <Camera color="#333" size={20} strokeWidth={1.5} />
      <ArrowRight />
    </div>
  );
}
```

Props map onto SVG attributes: `size`, `color`, `strokeWidth`, `absoluteStrokeWidth`, plus any pass-through SVG prop. The vanilla `lucide` package instead exposes a `createIcons()` DOM scan that replaces `<i data-lucide="camera">` placeholders.

## Architecture / How It Works

Source of truth is `icons/*.svg` — one file per icon, hand-authored against a documented spec: 24×24 viewBox, 2px stroke, `stroke-linecap="round"`, `stroke-linejoin="round"`, and alignment to a 1px grid[^5]. Each SVG has a companion JSON metadata file holding tags, categories, aliases, and contributor credit; this metadata drives search on lucide.dev and the alias system.

The framework packages are generated, not written. A build step reads the shared SVG set and emits per-framework component files — this is why a new icon lands simultaneously across React, Vue, Svelte, Solid, and the rest without per-framework hand edits. Each generated icon is a separate ES module re-exported from a barrel `index.js`. That structure is the whole tree-shaking story: `import { Camera } from "lucide-react"` resolves to one small module, and a production bundler drops the other ~1600.

Newer framework packages use the scoped `@lucide/*` namespace (`@lucide/vue`, `@lucide/svelte`, `@lucide/angular`, `@lucide/astro`) while the original ones keep the unscoped `lucide-react` / `lucide-solid` / `lucide-preact` names[^3]. This split is historical, not a quality signal, but it trips up people who guess package names.

Aliases are first-class: when an icon is renamed, the old name is kept as an alias so imports don't break silently. Deprecated names emit warnings before eventual removal. This is the mechanism that lets the set evolve without constantly breaking downstream code — though see Production Notes for where it still bites.

## Production Notes

**The dev-server slowdown is the classic footgun.** Because every icon is a separate module behind a barrel export, a naive `import { X } from "lucide-react"` forces some dev toolchains (notably Vite's dependency pre-bundling) to crawl thousands of modules, producing multi-second cold reloads and huge dev module graphs. Production builds are fine (tree-shaking removes the rest); the pain is dev-time only. Mitigations: let Vite pre-bundle `lucide-react` via `optimizeDeps.include`, use deep per-icon imports (`lucide-react/dist/esm/icons/camera`), or the dynamic-import map for icon-name-driven rendering. This surfaces repeatedly in issues and is the single most common Lucide complaint.

**Rendering icons by dynamic name is not free.** If you look an icon up from a runtime string (e.g. a CMS field), you cannot tree-shake — you either bundle the whole set or use the provided `dynamicIconImports` / lazy map so each icon code-splits on demand. Plan for this at design time; retrofitting it is annoying.

**Icon churn across versions.** Icons are added, renamed, and occasionally removed. Aliases soften renames, but a pinned icon name can still disappear on a major bump, and new icons mean the "latest" set differs between your machine and a teammate's if versions drift. Pin the package version and read release notes before upgrading if you depend on specific glyphs.

**No brand logos, ever.** If you need GitHub/Google/Discord marks, Lucide will not have them and will not add them[^4]. Pair it with a dedicated brand-icon set (e.g. Simple Icons) rather than filing requests.

**Licensing label mismatch.** GitHub's automated license detector reports `NOASSERTION` for this repo, but the actual `LICENSE` file and lucide.dev both state the ISC License — permissive, commercial-use OK[^6]. The `NOASSERTION` is a detector quirk (the file mixes Lucide's ISC grant with the inherited Feather MIT notice), not an ambiguity you need to worry about for normal use.

## When to Use / When Not

**Use when:**
- You want a large, actively maintained, visually consistent stroke-icon set across one or many frameworks.
- You care about bundle size and can import icons individually.
- You value a stable design language (uniform 2px stroke, 24px grid) over per-icon expressiveness.

**Avoid when:**
- You need brand/company logos — Lucide categorically excludes them.
- You need filled or multi-color/duotone icons — Lucide is stroke-only by design.
- You render icons purely from runtime strings and can't afford the dynamic-import setup or full-set bundle.
- You want a frozen, never-changing name set — icon churn is a real (if managed) cost.

## Alternatives

- tabler/tabler-icons — larger stroke-icon set with a similar aesthetic; use when you want even more coverage in one consistent style.
- feathericons/feather — the original Lucide forked from; use only for legacy parity, as it is far less active.
- simple-icons/simple-icons — brand/company logos specifically; use alongside Lucide to fill the gap Lucide refuses.
- iconify/iconify — an aggregator serving 200k+ icons from many sets (including Lucide) via one API; use when you want many icon families behind a single runtime.
- Radix Icons / heroicons — smaller, opinionated sets tied to a design system; use when you want fewer, curated glyphs matching Radix or Tailwind UI.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2020-06 | Repo created as a community fork of Feather Icons[^1]. |
| — | 2020–2022 | Icon set expands well beyond Feather; per-framework packages added. |
| — | 2023–2024 | Scoped `@lucide/*` packages introduced for newer frameworks; alias/deprecation system matures. |
| — | 2026-07 | ~23.4k stars, 1.4k forks, 1600+ icons; pushed daily[^2]. |

Lucide does not headline a single semantic version across the monorepo; each framework package versions independently on npm, so "version" is best read per-package rather than repo-wide.

## References

[^1]: Lucide is a fork of Feather Icons; project background and rationale. https://lucide.dev/guide/
[^2]: Repository metadata via GitHub API, 2026-07-15: 23,447 stars, 1,458 forks, last push 2026-07-14. https://github.com/lucide-icons/lucide
[^3]: Lucide packages list (React, Vue, Svelte, Solid, Preact, Angular, Astro, React Native, vanilla, static). https://lucide.dev/packages
[^4]: "About brand logos" — Lucide's official statement on excluding brand logos. https://github.com/lucide-icons/lucide/blob/main/BRAND_LOGOS_STATEMENT.md
[^5]: Lucide icon design guidelines (24×24 grid, 2px round stroke). https://lucide.dev/guide/design/icon-design-guide
[^6]: ISC License per repo LICENSE and lucide.dev; GitHub's detector reports NOASSERTION. https://github.com/lucide-icons/lucide/blob/main/LICENSE

## Tags

typescript, icons, svg, ui-components, react, vue, svelte, tree-shaking, feather-fork, design-system, frontend, icon-library
