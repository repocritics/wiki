# react-icons/react-icons

> SVG icon components for React, bundling ~30 popular open-source icon packs behind per-pack ES module imports.

[GitHub repo](https://github.com/react-icons/react-icons) ·
[Official website](https://react-icons.github.io/react-icons/) ·
License: MIT (wrapper code) — see licensing note below

## Overview

react-icons is a single npm package that re-packages the SVGs of many separate icon projects — Font Awesome, Material Design, Bootstrap Icons, Feather, Lucide, Heroicons, Phosphor, Tabler, Simple Icons, and roughly twenty more — as React components with a uniform API[^1]. Instead of pulling each icon project as its own dependency with its own conventions, you import `{ FaBeer } from "react-icons/fa"` and get a component that takes `size`, `color`, `className`, and standard SVG props. It has been maintained since 2015 and is one of the most-depended-on icon packages in the React ecosystem.

The defining design choice is the **per-pack subpath**: every icon set lives under its own import path (`react-icons/fa`, `react-icons/md`, `react-icons/lu`, …), and each icon is a named export from that subpath. This is what makes tree-shaking possible — a production bundler drops the thousands of icons you didn't reference. It is also the source of the project's single most common complaint: the per-pack barrel files are enormous, and toolchains that don't tree-shake (or that eagerly evaluate barrels in dev) pay for it. See Production Notes.

The defining tension is **breadth vs. freshness**. react-icons is a mirror, not a source: the icons are fetched from upstream projects at build time and frozen at whatever version the maintainers last vendored[^2]. You get one consistent API across thirty icon families, but you do not get the latest icons from any single family until react-icons re-fetches and cuts a release. Teams that need the current Lucide or Font Awesome set are often better served by that project's own React package.

## Getting Started

```bash
npm install react-icons
# or: yarn add react-icons / pnpm add react-icons
```

```jsx
import { FaBeer } from "react-icons/fa";      // Font Awesome 5
import { MdHome } from "react-icons/md";      // Material Design
import { LuSettings } from "react-icons/lu";  // Lucide

function Toolbar() {
  return (
    <nav>
      <MdHome size={24} color="#333" />
      <LuSettings size={24} />
      <FaBeer />                                {/* defaults to 1em, currentColor */}
    </nav>
  );
}
```

Global defaults are set through React context rather than per-icon props[^1]:

```jsx
import { IconContext } from "react-icons";

<IconContext.Provider value={{ color: "blue", size: "1.5em", className: "app-icon" }}>
  <MdHome />
</IconContext.Provider>;
```

## Architecture / How It Works

react-icons is a **code-generation monorepo**, not a hand-written component library. The `packages/react-icons` workspace holds a manifest (`src/icons/index.ts`) listing each icon set and the upstream Git ref to pull. `yarn fetch` clones the source SVGs, `yarn build` transforms every SVG into a data description and emits per-pack module files[^2]. Contributors do not commit thousands of `.tsx` files; they edit the manifest and regenerate.

At runtime each icon is a thin function component. The generated data (the SVG's viewBox and path/element tree) is passed to a shared renderer — `GenIcon` / `IconBase` — which builds the actual `<svg>` element, merges `IconContext` defaults with local props, applies `1em` sizing and `currentColor` fill by default, and wires the optional `title` for accessibility. Because every icon shares this renderer, the API is identical across all packs; only the geometry differs.

The subpath layout (`react-icons/fa`, `/md`, `/io5`, …) is the whole tree-shaking story. Each subpath is a barrel that re-exports every icon in that family by name. Modern bundlers doing dead-code elimination keep only the named imports you use. The two-letter prefix in each icon name encodes its pack (`Fa*`, `Md*`, `Io*`, `Bs*`, `Lu*`, `Pi*`, `Tb*`, `Si*`, …), which also prevents name collisions between packs.

There is a second distribution, `@react-icons/all-files`, that publishes one file per icon so you can deep-import a single icon (`@react-icons/all-files/fa/FaBeer`) on toolchains with weak tree-shaking. It trades a very large install for guaranteed minimal bundles — and it has not seen a fresh release in a long time, so its icon coverage lags the main package[^3].

## Production Notes

**The barrel-import bundle footgun.** `import { FaBeer } from "react-icons/fa"` names one icon but references a module that re-exports thousands. In a production build with tree-shaking this is fine; in environments that don't eliminate dead code — some Jest setups, older Webpack configs, non-tree-shaking SSR paths — you can accidentally pull an entire pack into the bundle. This is the longest-running issue in the tracker[^4]. Mitigations: rely on a modern bundler (Vite/Rollup, esbuild, Webpack 5 in production mode), or use `@react-icons/all-files` deep imports, or restrict yourself to a small number of packs.

**Dev-server slowness with Vite.** Even when production output is fine, Vite's dev server can be sluggish the first time it touches a react-icons subpath because it has to pre-bundle a barrel exporting thousands of exports. Adding the used subpaths to `optimizeDeps.include`, or importing from `@react-icons/all-files`, reduces the cost.

**Icons are frozen at vendored versions.** The README pins each pack to a specific upstream version/ref (e.g. Font Awesome 6.5.2, Lucide a mid-5.x snapshot, Tabler 3.2.0, Phosphor 2.1.1)[^1]. Newly added upstream icons are unavailable until react-icons re-fetches and releases. If you depend on the newest icons from one family, use that family's own package.

**Licensing is not one license.** GitHub reports the repository license as `NOASSERTION` because it is a mixture: the react-icons wrapper code is MIT, but each bundled icon set carries its own terms — MIT, Apache-2.0, ISC, CC BY 4.0, CC BY-SA 3.0, CC0, SIL OFL 1.1, MPL-2.0[^1]. Several (the CC BY families, weather icons under OFL) impose attribution or share-alike obligations on the icons themselves. Shipping react-icons means auditing which packs you actually use and honoring their licenses; you cannot treat the whole thing as "MIT."

**v2 → v3 removed default vertical alignment.** Before v3, icons received `vertical-align: middle` automatically; from v3 on they do not, so upgraded apps often show icons sitting on the text baseline. The fix is to set it globally via `IconContext` (`style: { verticalAlign: "middle" }`) or a shared `className`[^1].

**No named-export type errors until build.** Because subpaths are generated, a typo like `FaBeerr` fails as a missing export rather than a helpful suggestion. Editors with the bundled TypeScript types autocomplete correctly (no `@types/react-icons` needed since v3), but mistyped icon names surface at import/build time.

## When to Use / When Not

**Use when:**
- You want icons from several different families behind one consistent React API.
- You're prototyping or building a general app UI and don't need the bleeding-edge version of any single icon set.
- You want `size`/`color`/`currentColor` behavior and context-based defaults without wiring an SVG loader yourself.

**Avoid when:**
- You depend on one icon family and need its latest icons — use its first-party React package (`lucide-react`, `@fortawesome/react-fontawesome`, `@heroicons/react`, `@phosphor-icons/react`).
- Your build pipeline can't tree-shake and bundle size is critical — the barrel imports will bite you.
- You need custom or design-system icons — react-icons only mirrors upstream open-source sets; bring your own SVG pipeline (SVGR) instead.
- Attribution/share-alike license obligations are a blocker and you can't audit per-pack terms.

## Alternatives

- lucide-icons/lucide (`lucide-react`) — use instead when you've standardized on one clean, actively updated icon set and want its newest glyphs.
- FortAwesome/react-fontawesome — use instead when you're committed to Font Awesome and want its official React integration, including Pro icons.
- tailwindlabs/heroicons (`@heroicons/react`) — use instead when you want the small, Tailwind-aligned Heroicons set at its current version.
- gregberge/svgr — use instead when your icons are your own SVGs and you want to compile them to components yourself rather than consume a bundle.
- iconify/iconify — use instead when you want on-demand access to a hundred-plus icon sets loaded at runtime/build without vendoring each one.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial repo | 2015-10 | Project created; wraps upstream icon SVGs as React components[^5]. |
| 3.x | ~2018 | Major rewrite: per-pack subpath imports, ES6 tree-shaking, native TypeScript types, `vertical-align` default removed[^1]. |
| 4.x | ~2021 | More icon packs added; continued per-pack generation model. |
| 5.x | ~2024 | Additional sets (e.g. Lucide, Phosphor, expanded Font Awesome 6); current major line[^1]. |

## References

[^1]: react-icons README — installation, per-pack import model, `IconContext`, bundled pack list with versions and licenses, v2→v3 migration guide. https://github.com/react-icons/react-icons/blob/master/README.md
[^2]: react-icons build process — `packages/react-icons/src/icons/index.ts` manifest plus `yarn fetch` / `yarn build` code generation from upstream SVGs. https://github.com/react-icons/react-icons/blob/master/packages/react-icons/src/icons/index.ts
[^3]: `@react-icons/all-files` deep-import distribution and release-staleness note. https://github.com/react-icons/react-icons/issues/593
[^4]: Bundle-size / barrel-import discussion. https://github.com/react-icons/react-icons/issues/154
[^5]: GitHub repository metadata (created 2015-10-23). https://github.com/react-icons/react-icons

## Tags

react, icons, svg, typescript, ui-components, frontend, icon-library, tree-shaking, javascript, npm-package
