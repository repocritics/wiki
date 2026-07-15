# formatjs/formatjs

> The monorepo behind react-intl — ICU MessageFormat parsing, the Intl polyfills, and the extract/compile toolchain for JavaScript internationalization.

[GitHub repo](https://github.com/formatjs/formatjs) ·
[Official website](https://formatjs.github.io/) ·
License: BSD-3-Clause (per-package)

## Overview

FormatJS is not a single library but a monorepo of ~20 packages that together
implement internationalization for JavaScript around the ICU MessageFormat
standard[^1]. The best-known package is `react-intl`, the React binding layer;
the core is `intl-messageformat` and `@formatjs/icu-messageformat-parser`, which
parse and format ICU strings. The rest of the repo is a set of spec-compliant
polyfills for the ECMAScript `Intl` object (`NumberFormat`, `DateTimeFormat`,
`PluralRules`, `RelativeTimeFormat`, `ListFormat`, `DisplayNames`, `Locale`,
`Segmenter`) plus a build-time toolchain (`@formatjs/cli`, `babel-plugin-formatjs`,
`@formatjs/ts-transformer`, `eslint-plugin-formatjs`).

react-intl originated at Yahoo and its copyright still reads Oath Inc.[^2]; the
project was later consolidated under the FormatJS org and rewritten in TypeScript.
The defining design choice is that messages are authored in ICU MessageFormat —
`{count, plural, one {# item} other {# items}}` — rather than in a bespoke
interpolation syntax. This buys correct pluralization, gender, and nested
selects for every locale CLDR knows about, at the cost of a heavier runtime and
a mandatory extract/translate/compile pipeline.

The project's relevance has shifted over its lifetime. When it started, browsers
and Node had little or no native `Intl`, so the polyfills were essential. Modern
runtimes now ship most of `Intl` natively, so much of the polyfill surface is
optional — the enduring value is the message pipeline, not the primitives.

## Getting Started

```bash
npm install react-intl
```

```tsx
import { IntlProvider, FormattedMessage, useIntl } from "react-intl";

const messages = {
  greeting: "Hello, {name}! You have {count, plural, one {# message} other {# messages}}.",
};

function Inbox({ name, count }: { name: string; count: number }) {
  const intl = useIntl();
  // Declarative form:
  return (
    <FormattedMessage
      id="greeting"
      defaultMessage={messages.greeting}
      values={{ name, count }}
    />
  );
}

export default function App() {
  return (
    <IntlProvider locale="en" messages={messages}>
      <Inbox name="Ada" count={2} />
    </IntlProvider>
  );
}
```

Message extraction and compilation are done with the CLI:

```bash
npx formatjs extract 'src/**/*.{ts,tsx}' --out-file lang/en.json
npx formatjs compile lang/fr.json --out-file compiled/fr.json
```

## Architecture / How It Works

The data flow is: **author → parse → format**, with an optional **extract →
compile** build step layered on top.

- **Parsing.** `@formatjs/icu-messageformat-parser` turns an ICU string into an
  AST (literals, argument placeholders, plural/select nodes, tags). This parse
  is the expensive part.
- **Formatting.** `intl-messageformat` (`IntlMessageFormat`) takes that AST plus
  runtime `values` and delegates the actual number/date/plural work to the
  environment's `Intl` object — native where available, FormatJS polyfill where
  not.
- **React layer.** `react-intl` wraps the above in `<IntlProvider>`, the
  `useIntl()` hook, and components like `<FormattedMessage>` /
  `<FormattedNumber>`. It memoizes formatters per locale to avoid re-parsing.

The extract/compile toolchain exists to move the parse cost out of the runtime.
`babel-plugin-formatjs` or `@formatjs/ts-transformer` walk your source at build
time, pull every `defineMessages` / `<FormattedMessage>` into a catalog keyed by
message ID, and can pre-compile translated catalogs into the parser AST so the
browser ships JSON that is already parsed. This is why `react-intl` insists on
stable message IDs: they are the join key between source, translation, and
compiled output.

The polyfill packages are independent and dependency-light. Each mirrors one
`Intl` constructor and loads its CLDR locale data separately — you import the
polyfill, then import the locale-data file for each locale you support.

## Production Notes

**Locale data is the main footgun.** The polyfills and formatters need CLDR data
per locale, loaded explicitly. The naive path — importing all locale data — adds
hundreds of KB to the bundle. Load only the locales you ship, and prefer native
`Intl` (skip the polyfill entirely) on runtimes that already implement it.

**Compile in production, parse in dev.** Shipping raw ICU strings means the
parser runs in the browser on every message. For anything beyond a handful of
strings, run `formatjs compile` so the client receives pre-parsed AST. Skipping
this is the most common performance complaint.

**Message ID discipline.** Without the extraction toolchain wired into the build,
catalogs drift from source: orphaned translations, missing IDs, silent fallback
to `defaultMessage`. Adopt `eslint-plugin-formatjs` to enforce that every message
has an ID and valid ICU syntax at lint time.

**The v3 rewrite (2019) was a hard break.** react-intl v3 was a TypeScript
rewrite that changed the imperative API, formatter memoization, and dropped
several v2 patterns[^3]. Upgrades across v2→v3 required non-trivial migration;
later majors (v4, v5, v6) were smaller but each tightened peer-dependency ranges
on React.

**Bundle weight.** `react-intl` + `intl-messageformat` + the parser is not tiny.
Teams that only need number/date formatting are often better served by native
`Intl`; FormatJS earns its size when you have translated catalogs with real
pluralization and interpolation.

## When to Use / When Not

**Use when:**
- You have a React app with translated message catalogs and need correct
  pluralization, gender, and nested selects across many locales.
- You want a standards-based (ICU MessageFormat) authoring format that
  translators and TMS tools already understand.
- You need a build-time extraction/compilation pipeline rather than hand-managed
  translation files.

**Avoid when:**
- You only need to format numbers, dates, or relative times — call native `Intl`
  directly and skip the dependency.
- You are not on React and want a batteries-included framework: i18next covers
  more ground for framework-agnostic setups.
- You want the smallest possible runtime and are willing to use macros/compile
  everything away — Lingui targets that niche with the same ICU syntax.

## Alternatives

- i18next/i18next — framework-agnostic i18n with its own plural/interpolation
  system and a large plugin ecosystem; use when you need backends, detection, and
  namespaces more than strict ICU compliance.
- lingui/js-lingui — also ICU MessageFormat with compile-time extraction via
  macros; use when you want a lighter runtime and are comfortable with a build
  step doing more of the work.
- Native `Intl` (ECMAScript) — use when you only format numbers/dates/lists and
  have no translated message strings; zero dependencies.
- intlify/vue-i18n — the equivalent stack for Vue; use instead of react-intl on
  Vue projects.
- angular/angular (`@angular/localize`) — Angular's built-in i18n; use within the
  Angular toolchain rather than bolting on react-intl.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2014-04-24 | FormatJS org / react-intl origins at Yahoo[^2]. |
| react-intl v2 | 2016 | ICU MessageFormat components, `injectIntl` HOC. |
| react-intl v3 | 2019 | TypeScript rewrite, hooks (`useIntl`), imperative API changes[^3]. |
| react-intl v4 | 2020 | React 16.3+ context, formatter cleanup. |
| react-intl v5 | 2020 | `@formatjs/*` polyfill consolidation, ESM output. |
| react-intl v6 | 2022 | React 18 support, updated Intl polyfills. |

## References

[^1]: Unicode ICU MessageFormat specification. https://unicode-org.github.io/icu/userguide/format_parse/messages/
[^2]: react-intl `LICENSE.md` — "Copyright 2019 Oath Inc." (BSD-3-Clause), reflecting the project's Yahoo/Oath origin. https://github.com/formatjs/formatjs/blob/main/packages/react-intl/LICENSE.md
[^3]: FormatJS docs, react-intl upgrade guides. https://formatjs.github.io/docs/react-intl/upgrade-guide-3x/

## Tags

typescript, javascript, i18n, internationalization, localization, react, react-intl, icu-messageformat, intl-polyfill, monorepo, translation, formatting
