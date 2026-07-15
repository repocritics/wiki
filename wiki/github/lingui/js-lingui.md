# lingui/js-lingui

> A compile-time internationalization framework for JavaScript: write messages as natural JSX or template literals, and a macro extracts them into ICU MessageFormat catalogs.

[GitHub repo](https://github.com/lingui/js-lingui) ·
[Official website](https://lingui.dev) ·
[License: MIT](https://github.com/lingui/js-lingui/blob/main/LICENSE)

## Overview

Lingui is an i18n toolchain built around a build-time macro. Instead of referencing translations by opaque keys, you wrap human-readable text in a `Trans` component or a `t` template tag, and a Babel or SWC macro rewrites that source into ICU MessageFormat strings plus runtime calls at compile time[^1]. The message id can be the source text itself or an explicit id — either way the developer reads real sentences in the code, not `t("home.header.title")`. The runtime that ships to the browser is small (the core is advertised at roughly 2 kb) because the heavy lifting — extraction, ID hashing, ICU parsing — happens during the build and the `compile` step.

The project splits into layers: `@lingui/core` is framework-agnostic and works in any JS project, `@lingui/react` adds components (including React Server Components support), and `@lingui/solid` provides SolidJS bindings; React Native uses the same extract-and-compile flow, while Astro and Svelte are covered by community packages[^2]. Tooling — the CLI, a Vite plugin, and an ESLint plugin — drives the extract → translate → compile pipeline. Catalogs default to gettext PO files, which most translation platforms already understand, with CSV/JSON and custom formatters available.

The defining tradeoff is the compile step. Lingui's clean, key-less authoring is only possible because a compiler transforms your code; that means Lingui is coupled to your build tooling (a Babel plugin, the SWC plugin, or the Vite plugin), the extracted catalog can drift from source if `extract` is not re-run, and the macro's transformation is not always obvious when debugging. Teams that want a plain runtime API with no build-time magic tend to prefer react-intl or i18next instead.

## Getting Started

```bash
npm install --save @lingui/core @lingui/react
npm install --save-dev @lingui/cli @lingui/babel-plugin-lingui-macro
```

```js
// lingui.config.js
export default {
  locales: ["en", "cs", "fr"],
  catalogs: [
    { path: "src/locales/{locale}/messages", include: ["src"] },
  ],
}
```

```jsx
// App.jsx — macro rewrites <Trans> into an ICU message at build time
import { Trans } from "@lingui/react/macro"
import { i18n } from "@lingui/core"
import { I18nProvider } from "@lingui/react"
import { messages } from "./locales/en/messages"   // produced by `lingui compile`

i18n.load("en", messages)
i18n.activate("en")

function App() {
  return (
    <I18nProvider i18n={i18n}>
      <Trans>Read the <a href="https://lingui.dev">documentation</a>.</Trans>
    </I18nProvider>
  )
}
```

```bash
npx lingui extract   # scan source, write/update PO catalogs
npx lingui compile   # PO -> optimized JS the runtime imports
```

## Architecture / How It Works

The pipeline has three stages, and understanding the boundary between them is the key to using Lingui well:

1. **Macro transform (build time).** `@lingui/core/macro` (the `t`, `plural`, `select` tags) and `@lingui/react/macro` (the `Trans`, `Plural` components) are macros, not real runtime imports. A Babel plugin or the SWC plugin replaces them with plain `i18n._(...)` calls and an ICU message string. If the macro is not wired into your build, the imports resolve to nothing useful and messages are never extracted.
2. **Extraction (`lingui extract`).** The CLI statically analyzes source to collect every message, computes an id (a hash of the source text, or your explicit id), and writes catalogs — PO by default. This is the step that keeps catalogs in sync with code; skipping it is the most common cause of "my new string isn't translated."
3. **Compile (`lingui compile`).** PO catalogs are turned into optimized JavaScript modules where ICU messages are pre-parsed into an efficient form. The app imports these compiled modules, not the PO files. This is why the shipped runtime stays small — it never parses raw ICU or PO at request time.

At runtime, `@lingui/core`'s `i18n` object holds the active locale and loaded catalog; `i18n._(id, values)` looks up the compiled message and formats it with `Intl` primitives (numbers, dates, plurals). `@lingui/react` wraps this in an `I18nProvider` context so components re-render on `i18n.activate()`. Rich-text messages work because `Trans` can carry JSX children, which are serialized to indexed placeholders (`<0>...</0>`) in the catalog and re-hydrated with the original elements at render.

Plurals, `select`, and ordinal formatting are standard ICU MessageFormat, so message strings are portable to any ICU-aware tool and are close enough to react-intl's format that migration is mechanical.

## Production Notes

- **The extract/compile steps are a real workflow obligation.** CI should run `lingui extract` (or at least fail if catalogs are stale) and `lingui compile` before the app build. A missing `compile` yields runtime errors or untranslated fallbacks; a missing `extract` silently ships out-of-date catalogs.
- **Macro tooling must match your bundler.** Babel projects need the Lingui Babel macro; SWC-based stacks (Next.js, etc.) need the SWC plugin; Vite users typically use the Vite plugin so catalogs compile on the fly. Mismatched or absent macro configuration is the single most frequent setup failure — the code compiles but no messages are extracted.
- **RSC support is comparatively new.** React Server Components support exists in `@lingui/react`, but server/client boundary rules (where `I18nProvider` lives, which macro import to use) are less battle-tested than the classic client-side path; validate on your framework version.
- **Message id strategy is a one-way-ish decision.** Using source text as the id keeps code readable but means editing the English copy changes the id and orphans existing translations; explicit ids decouple copy edits from translation identity at the cost of readability. Choose deliberately before catalogs grow large.
- **SWC plugin/version coupling.** SWC plugins are pinned to specific SWC ABI versions, so the Lingui SWC plugin can lag or conflict during toolchain upgrades — a known friction point for Next.js users on the SWC path.
- **v5/v6 macro import paths changed.** Newer versions moved macros to `@lingui/core/macro` and `@lingui/react/macro` (rather than a single `@lingui/macro`); upgrade guides matter because the import paths and macro packages were reorganized[^3].

## When to Use / When Not

**Use when:**
- You want translatable source that reads as real sentences and are willing to run a build-time extract/compile step.
- You need proper ICU plural/select/number/date handling and standard PO catalogs your translators already use.
- You are on React (including RSC), SolidJS, or React Native and want first-class components plus CLI/Vite/ESLint tooling.
- You are migrating off react-intl and want a similar message format with cleaner authoring.

**Avoid when:**
- You cannot add a Babel/SWC/Vite macro to your build, or you want a pure-runtime library with no compile step.
- You need translations loaded and swapped entirely at runtime from a remote source with no rebuild — i18next's model fits better.
- Your stack is outside the first-class targets (e.g. Vue), where other libraries are the native choice.
- Your app has only a handful of strings and the extract/compile pipeline is more ceremony than the problem warrants.

## Alternatives

- i18next/i18next — use instead when you want the largest ecosystem, runtime/remote catalog loading, and no build-time extraction, across React and non-React stacks.
- formatjs/formatjs — react-intl; use when you prefer an explicit, mature runtime API with the same ICU MessageFormat but without Lingui's macro/compile step.
- projectfluent/fluent.js — use when you want Mozilla's Fluent syntax (asymmetric, translator-driven grammar) rather than ICU MessageFormat.
- intlify/vue-i18n — use for Vue applications, where Lingui has no first-class binding.
- nextjs built-in i18n routing — use when you only need locale-prefixed routing and will supply your own message layer.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | 2017 | Initial public releases; created by Tomáš Ehrlich[^1]. |
| 2.x | 2018 | Macro-based authoring and ICU MessageFormat catalogs. |
| 3.x | 2020 | Core/react split matured; reworked macro and CLI. |
| 4.x | 2023 | SWC plugin support alongside Babel; tooling modernization. |
| 5.x | 2024–2025 | React Server Components support; macro packages reorganized. |
| 6.0 | 2026-04-22 | Latest major line; SolidJS bindings, AI-workflow tooling[^2][^3]. |

## References

[^1]: js-lingui documentation and repository. https://lingui.dev and https://github.com/lingui/js-lingui
[^2]: js-lingui README, "Key Features" (universal core/react/solid, RSC, React Native; community Astro/Svelte). https://github.com/lingui/js-lingui#readme
[^3]: "Announcing Lingui 6.0" release post, 2026-04-22. https://lingui.dev/blog/2026/04/22/announcing-lingui-6.0

## Tags

javascript, typescript, i18n, internationalization, localization, react, icu-messageformat, macros, react-server-components, gettext-po, solidjs, react-native
