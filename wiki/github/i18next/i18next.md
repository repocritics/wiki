# i18next/i18next

> Framework-agnostic JavaScript internationalization core — a small runtime and a large plugin ecosystem around it.

[GitHub repo](https://github.com/i18next/i18next) ·
[Official website](https://www.i18next.com) ·
[License: MIT](https://github.com/i18next/i18next/blob/master/LICENSE)

## Overview

i18next is an internationalization (i18n) library for JavaScript, first
published in 2011[^1] and still actively maintained (last push July 2026,
~8.6k stars). Its scope is deliberately narrow: it provides a `t()`
translation function with interpolation, pluralization, context, and
namespaces, plus a plugin contract. Everything else — loading translation
files, detecting the user's language, framework bindings — lives in separate
packages. "i18next core" is a few tens of kilobytes; the thing people call
"i18next" in practice is usually core plus three or four plugins.

The framework is runtime-agnostic (browser, Node.js, Deno, React Native) and
UI-agnostic. The React binding (`react-i18next`), Vue, Angular, and Svelte
integrations are maintained by the same organization but versioned
independently, which is the main source of confusion for newcomers: the
GitHub repo here is *only* the core, and most application-facing behavior
(the `useTranslation` hook, SSR helpers, Next.js glue) lives elsewhere.

The project's defining tension is commercial. i18next is genuinely free MIT
software, but it is developed by the makers of Locize, a paid
translation-management SaaS, and the README and docs steer heavily toward
it[^2]. None of the core features require Locize — file-based and
custom-backend setups are fully supported — but readers should treat the
"official service" framing as vendor marketing layered on top of an
otherwise neutral OSS library.

## Getting Started

```bash
npm install i18next
# common companions:
npm install react-i18next i18next-http-backend i18next-browser-languagedetector
```

```js
import i18next from 'i18next';

await i18next.init({
  lng: 'en',
  fallbackLng: 'en',
  resources: {
    en: { translation: {
      greeting: 'Hello {{name}}',
      items_one: '{{count}} item',
      items_other: '{{count}} items',
    } },
    de: { translation: { greeting: 'Hallo {{name}}' } },
  },
});

i18next.t('greeting', { name: 'Ada' }); // "Hello Ada"
i18next.t('items', { count: 3 });       // "3 items"
i18next.changeLanguage('de');           // returns a Promise
```

The `_one` / `_other` key suffixes are the modern plural form (see below);
older tutorials using `_plural` predate a breaking change and no longer work.

## Architecture / How It Works

The core is an event-emitting singleton (you can also construct isolated
instances via `i18next.createInstance()`). Translations are looked up by a
key path within a **namespace** — the unit of lazy-loading and code-splitting
for strings. Missing keys fall back through a configurable chain
(`fallbackLng`, `fallbackNS`, parent language like `en` for `en-US`).

Extensibility is a small set of plugin *types*, each registered with
`i18next.use(...)`:

- **Backend** — loads translation resources (`i18next-http-backend`,
  `i18next-fs-backend`, `i18next-locize-backend`). Without one, resources must
  be passed inline.
- **LanguageDetector** — resolves the active language from cookie, header,
  querystring, `navigator`, etc. (`i18next-browser-languagedetector`).
- **Post-processor** — transforms a resolved string (e.g. `sprintf`,
  interval plurals).
- **i18nFormat / formatter** — integrates ICU MessageFormat or custom
  formatting (`i18next-icu`).

Interpolation is `{{var}}` by default with HTML-escaping on. Pluralization
is the internals story that bites people: since **v21 (2021)** i18next
delegates to the platform `Intl.PluralRules`, so plural keys use CLDR
categories (`_zero`, `_one`, `_two`, `_few`, `_many`, `_other`) instead of
the old `_plural` / numeric-suffix scheme[^3]. This shipped alongside the
"JSON v4" resource format and is the single most common upgrade break in the
ecosystem.

Nesting (`$t(otherKey)`) and context (`key_male` / `key_female` via a
`context` option) are resolved inside the core lookup, before
post-processing. The core does not do date/number formatting itself beyond a
minimal `format` hook — locale-aware number and date formatting is expected
to come from `Intl` or the ICU plugin.

## Production Notes

**Bundle weight vs. alternatives.** Core plus the usual React + backend +
detector plugins is meaningfully larger than minimal libraries like
polyglot.js. For static, few-locale sites this is often more machinery than
needed; the payoff is realized when you use namespaces, lazy loading, and
plural/context features. Tree-shaking helps but the plugin objects are mostly
runtime, not compile-time.

**Async init is a real footgun.** `init()` and `changeLanguage()` return
Promises and load resources asynchronously. Rendering before resources are
ready yields keys or fallback text. In React this manifests as flashes of
translation keys; `react-i18next` addresses it with `Suspense` or the
`ready` flag, but you must opt in. SSR frameworks need the resources loaded
on the server request path, which is why `next-i18next` / the App Router
helpers exist as separate packages.

**Version skew across the ecosystem.** Because bindings are versioned
independently, a `i18next` bump can outrun `react-i18next`,
`next-i18next`, or backend plugins' peer-dependency ranges. Pin and upgrade
the set together; read each companion's changelog, not just core's.

**The v20→v21 plural migration** requires rewriting plural keys and, for
some languages, adding categories you didn't have before. Tools exist to
convert JSON v3→v4, but translator workflows and any custom key-generation
scripts need updating too. Budget for it rather than treating it as a patch.

**Debugging.** Set `debug: true` and watch for `missingKey` events; the
`saveMissing` option can auto-report absent keys to a backend. Silent
fallback to the key string is the default, so missing translations do not
throw and are easy to ship unnoticed — wire up `missingKeyHandler` in CI.

## When to Use / When Not

**Use when:**
- You need plurals, context, nesting, and namespaces across many locales.
- You target multiple runtimes/frameworks and want one translation model.
- You want lazy-loaded translation bundles and a pluggable backend.
- You're already in the React/Vue ecosystem where the bindings are mature.

**Avoid when:**
- You have a small, mostly-static site with a handful of strings — polyglot.js
  or hand-rolled maps are lighter.
- You want strict ICU MessageFormat as the primary syntax — FormatJS or
  Lingui model that natively (i18next needs the ICU plugin).
- You want compile-time-extracted, type-safe catalogs — Lingui's macro/compile
  approach fits better than i18next's runtime keys.
- Bundle size is a hard budget and features are minimal.

## Alternatives

- formatjs/formatjs (react-intl) — ICU MessageFormat-first, standards-aligned;
  use when ICU syntax and formatting are central.
- lingui/js-lingui — compile-time extraction + ICU; use when you want type
  safety and smaller runtime catalogs.
- intlify/vue-i18n — the idiomatic choice inside Vue; use instead of wiring
  i18next into Vue.
- airbnb/polyglot.js — tiny interpolation + basic plurals; use for small apps
  that don't need the plugin ecosystem.
- angular/angular (@angular/localize) — built-in i18n for Angular; use when you
  stay within Angular's compile-time pipeline.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2011-12 | First release by Jan Mühlemann (jamuhl)[^1]. |
| 2.x | ~2015 | Core rewrite; plugin architecture consolidated. |
| 15–20 | 2018–2020 | Backend/detector split matured; namespaces, `saveMissing`. |
| 21.0 | 2021 | `Intl.PluralRules` pluralization + JSON v4 format (breaking)[^3]. |
| 22–24 | 2022–2024 | Incremental releases; formatter/ICU improvements. |

## References

[^1]: i18next repository, created 2011-12-16. https://github.com/i18next/i18next
[^2]: i18next README — "official service" / Locize promotion. https://github.com/i18next/i18next#readme
[^3]: i18next documentation, "Plurals" (Intl.PluralRules, JSON v4). https://www.i18next.com/translation-function/plurals

## Tags

javascript, typescript, i18n, internationalization, translation, localization, nodejs, deno, react, plugin-architecture, frontend
