# mui/material-ui

> A comprehensive React component library implementing Google's Material Design — the most-installed React UI kit, and a long-running case study in the cost of runtime CSS-in-JS.

[GitHub repo](https://github.com/mui/material-ui) ·
[Official website](https://mui.com/material-ui/) ·
[License: MIT](https://github.com/mui/material-ui/blob/master/LICENSE)

## Overview

Material UI is a library of ready-made, themeable React components — buttons, inputs, dialogs, tables, navigation — that implement Google's Material Design specification independently of Google[^1]. First released as `material-ui` in 2014, it predates most of the current React component-library field and is the most-depended-on of them by npm install count. The project is maintained by MUI, a commercial open-source company whose revenue comes from the paid tiers of MUI X and premium templates rather than from Material UI itself, which is MIT and, per the maintainers, "free forever."

Material UI's defining tension is styling. From v1 through v4 it used JSS; v5 (2021) switched the default styling engine to Emotion, a runtime CSS-in-JS library[^2]. That runtime is the source of both its ergonomics (the `sx` prop, `styled()`, dynamic theming) and its most-cited drawback: styles are computed in the browser on every render, which costs measurable hydration and render time and does not compose cleanly with React Server Components. The project's multi-year answer to this is Pigment CSS[^3], a build-time, zero-runtime extraction engine intended to replace Emotion — a migration that reframes the library's entire styling story and is not yet the default.

A second point of friction is Material Design itself. Material UI's components track Material Design 2 (`m2`); adoption of Material Design 3 ("Material You") has been slow, so teams wanting the current Google look often theme heavily or look elsewhere. The library is best understood as "a mature, comprehensive Material-flavored component set" rather than "the current Material Design reference implementation."

## Getting Started

```bash
npm install @mui/material @emotion/react @emotion/styled
```

```tsx
// App.tsx
import { createTheme, ThemeProvider } from "@mui/material/styles";
import Button from "@mui/material/Button";
import Stack from "@mui/material/Stack";

const theme = createTheme({
  palette: { mode: "light", primary: { main: "#1976d2" } },
});

export default function App() {
  return (
    <ThemeProvider theme={theme}>
      <Stack spacing={2} sx={{ p: 4, maxWidth: 320 }}>
        <Button variant="contained" onClick={() => alert("clicked")}>
          Primary
        </Button>
        <Button variant="outlined" sx={{ borderRadius: 2 }}>
          Custom radius via sx
        </Button>
      </Stack>
    </ThemeProvider>
  );
}
```

Prefer per-component imports (`@mui/material/Button`) over barrel imports (`import { Button } from "@mui/material"`) — the barrel can pull the whole package into some bundler/dev setups and slow cold builds.

## Architecture / How It Works

Material UI ships as several coordinated packages rather than one:

- **`@mui/material`** — the Material Design components.
- **`@mui/system`** — the style/theme engine: `sx`, `styled()`, the `createTheme` model, and the responsive/spacing primitives shared across MUI products.
- **`@mui/base` / Base UI** — unstyled, accessible "headless" component logic that Material UI is built on top of. Base UI has since spun out into its own project ([base-ui.com](https://base-ui.com)), a separate line from the Material components[^4].
- **`@mui/icons-material`** — the Material Icons set as tree-shakeable React components.
- **`@mui/lab`** — components incubating before promotion to the stable package.

**Theming** is a single JavaScript object (`createTheme`) consumed via React context (`ThemeProvider`). Components read theme tokens (palette, typography, spacing, breakpoints, shadows) at render time. v6 added a CSS-variables mode (`cssVariables: true`) that emits real CSS custom properties, which reduces SSR/dark-mode flicker and is a prerequisite for the zero-runtime direction[^5].

**Styling engine.** The default is Emotion; styled-components is supported as an alternative engine but with caveats. Both are *runtime* CSS-in-JS: the `styled()` factory and `sx` prop serialize styles and inject `<style>` tags as components mount. This is the mechanism behind theme-aware, prop-driven styling — and the reason a large Material UI tree does non-trivial style work on every render and during hydration.

**Pigment CSS** is the strategic response: a build-time transform (Babel/webpack/Vite plugins) that extracts `styled`/`sx` usage into static CSS at compile time, eliminating the browser runtime and making the components compatible with React Server Components[^3]. It is available but not the default path for `@mui/material`; treat it as an in-progress migration rather than a shipped default.

**MUI X** lives in a separate repository ([mui/mui-x](https://github.com/mui/mui-x)) and is where the complex components — Data Grid, Date/Time Pickers, Charts, Tree View — are developed. Its community tiers are MIT, but Pro and Premium features (advanced Data Grid, some chart features) require a commercial license and a runtime license key[^6]. This is the dividing line between the free core and the commercial product.

## Production Notes

**Runtime CSS-in-JS cost is real.** On large pages the Emotion runtime shows up in flame graphs as style serialization and injection during render and hydration. This is the single most common performance complaint. Mitigations: memoize component trees, avoid regenerating `sx` objects on every render (hoist static ones), lean on the CSS-variables theme mode, and watch for it on low-end mobile.

**React Server Components.** Runtime CSS-in-JS is fundamentally a client-side mechanism. In the Next.js App Router, Material UI components generally need to sit in client components (or use MUI's documented App Router integration), and full RSC/zero-runtime usage is what Pigment CSS is meant to unlock. Do not assume drop-in RSC support in the current default setup.

**Bundle size.** The full component set plus Emotion plus icons is heavy. Per-path imports and tree-shaking help, but `@mui/icons-material` in particular is enormous if imported carelessly — always import individual icons.

**Upgrades are the recurring tax.** The major migrations are non-trivial:
- **v4 → v5** is the big one: package rename (`@material-ui/*` → `@mui/*`), JSS → Emotion, `makeStyles`/`withStyles` deprecated in favor of `styled`/`sx`. MUI shipped codemods, but real apps with heavy `makeStyles` usage saw multi-week migrations.
- **v5 → v6** and **v6 → v7** are smaller but still carry breaking changes; v7 moved package internals toward ESM and adjusted the module layout[^7].

**Material Design version lag.** Components implement Material Design 2. If your design mandate is Material Design 3 / Material You, expect to theme extensively or reconsider the choice.

**Styling-engine mixing.** Choosing styled-components instead of Emotion, or mixing Material UI's `styled` with Tailwind, tends to create ordering/specificity issues (Material UI injects styles at runtime; static CSS may or may not win depending on injection order). The best-supported path is the default Emotion setup.

## When to Use / When Not

**Use when:**
- You want a large, mature, accessible component set out of the box and value breadth over a bespoke look.
- Your team is comfortable with a Material-flavored aesthetic (or will theme it heavily).
- You need complex data components (grid, pickers, charts) and are willing to pay for MUI X Pro/Premium.
- You want a decade of battle-testing and a large hiring pool who already know the API.

**Avoid when:**
- You are optimizing for minimal client runtime and want zero-runtime styling today (shadcn/ui + Tailwind, or Pigment-first stacks, fit better).
- You need current Material Design 3 fidelity without heavy custom theming.
- You want to fully own and edit component source rather than depend on a versioned package.
- Bundle size and hydration cost on constrained mobile devices are hard constraints.

## Alternatives

- shadcn-ui/ui — copy-paste components on Radix + Tailwind; you own the code and pay no CSS-in-JS runtime. Use when you want full control and static styling.
- ant-design/ant-design — comparably comprehensive, denser enterprise/desktop design language. Use when you want a batteries-included non-Material system.
- mantinedev/mantine — full component set plus hooks, modern API, not tied to Material Design. Use when you want breadth without the Material aesthetic.
- chakra-ui/chakra-ui — style-props ergonomics, lighter design opinion. Use when you want themeable components with a smaller conceptual surface.
- radix-ui/primitives — unstyled accessible primitives only. Use when you want accessibility behavior and will build the visual layer yourself.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2014–2018 | Original `material-ui`; pre-1.0 API, later served at v0.mui.com. |
| 1.0 | 2018-05 | First stable release; new component architecture[^1]. |
| 4.0 | 2019-05 | JSS-based styling (`makeStyles`/`withStyles`), hooks-era API. |
| 5.0 | 2021-09 | Package rename to `@mui/material`; Emotion default engine; `sx` prop; `styled()`[^2]. |
| 6.0 | 2024 | CSS-variables theming mode; SSR/dark-mode improvements[^5]. |
| 7.0 | 2025 | ESM-oriented package layout and module changes[^7]. |

## References

[^1]: Material UI — Google Material Design implementation. https://mui.com/material-ui/getting-started/
[^2]: Material UI v5 migration guide (JSS → Emotion, `@material-ui/*` → `@mui/*`). https://mui.com/material-ui/migration/migration-v4/
[^3]: Pigment CSS — zero-runtime CSS-in-JS extraction. https://github.com/mui/pigment-css
[^4]: Base UI — unstyled component library, spun out of MUI Base. https://base-ui.com
[^5]: Material UI theming — CSS theme variables. https://mui.com/material-ui/customization/css-theme-variables/overview/
[^6]: MUI X licensing (community MIT vs Pro/Premium commercial). https://mui.com/x/introduction/licensing/
[^7]: Material UI v7 migration guide. https://mui.com/material-ui/migration/upgrade-to-v7/

## Tags

react, component-library, design-system, material-design, css-in-js, frontend, ui-kit, javascript, typescript, emotion, accessibility
