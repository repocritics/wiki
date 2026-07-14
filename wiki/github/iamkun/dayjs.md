# iamkun/dayjs

> A 2 kB immutable date library that reimplements most of Moment.js's API surface, so migrating code barely changes and new code stays small.

[GitHub repo](https://github.com/iamkun/dayjs) ·
[Official website](https://day.js.org) ·
[License: MIT](https://github.com/iamkun/dayjs/blob/dev/LICENSE)

## Overview

Day.js is a small date-time library created by iamkun in 2018[^1], written to be a
near drop-in replacement for Moment.js. The core is roughly 2 kB minified+gzipped
and exposes a chainable API — `dayjs().add(1, 'day').format()` — that deliberately
mirrors Moment's method names and format tokens. The pitch is simple: if you know
Moment, you already know Day.js, but you ship a fraction of the bytes.

Two design decisions define it. First, **Day.js objects are immutable**: every
manipulation (`add`, `subtract`, `startOf`, `set`) returns a new instance rather
than mutating in place, which fixes Moment's most notorious footgun. Second, the
core is intentionally minimal — anything beyond parse/format/manipulate/query lives
in **opt-in plugins** loaded via `dayjs.extend()`, and locales are loaded on demand.
This keeps the default bundle tiny at the cost of making a lot of "expected" behavior
(custom parse formats, timezones, relative time, ISO week numbers) something you have
to remember to wire up.

The central tension is that Day.js is a thin wrapper over the native JavaScript
`Date`. That is why it is small, and also why it inherits `Date`'s limitations:
timezone handling leans on the environment's `Intl` implementation, and the
Moment-compatible token system is not the Unicode/CLDR standard. Day.js is an
excellent Moment migration target and a fine general-purpose date library, but it is
not the tool to reach for if you need rigorous calendar/timezone correctness.

## Getting Started

```console
npm install dayjs
```

```javascript
import dayjs from 'dayjs';

dayjs('2018-08-08')                       // parse an ISO string
  .add(1, 'year')                         // returns a NEW instance (immutable)
  .startOf('month')
  .format('YYYY-MM-DD HH:mm:ss');         // '2019-08-01 00:00:00'

dayjs().isBefore(dayjs().add(1, 'day'));  // query -> true
```

Plugins and locales are separate imports and must be registered before use:

```javascript
import dayjs from 'dayjs';
import customParseFormat from 'dayjs/plugin/customParseFormat';
import 'dayjs/locale/es';

dayjs.extend(customParseFormat);
dayjs('08/26/2024', 'MM/DD/YYYY');        // non-ISO parsing needs the plugin
dayjs().locale('es').format('dddd');      // 'sábado'
```

## Architecture / How It Works

A `Dayjs` instance holds a single native `Date` internally and treats it as
immutable. Chainable methods clone the underlying value, apply the change, and
return a fresh wrapper. This is the whole model — there is no custom calendar
engine, no big numeric time representation of its own. Localized month/day names,
timezone offsets, and DST transitions ultimately come from `Date` and, for the
timezone plugin, from the runtime's `Intl.DateTimeFormat`.

The **plugin system** is the other structural piece. `dayjs.extend(plugin)` runs a
function that receives the option object, the `Dayjs` class, and the `dayjs`
factory, letting it monkey-patch prototype methods or add statics. Notable official
plugins: `customParseFormat` (parse arbitrary token strings), `utc` and `timezone`
(the latter wraps `Intl` to convert offsets), `relativeTime` ("3 hours ago"),
`advancedFormat`, `isoWeek`, `duration`, and `weekday`. Because plugins patch a
shared prototype, load order and global registration matter — extending the same
capability twice, or relying on a plugin another module registered, is a common
source of "works in one file, breaks in another" bugs.

Locales follow the same load-on-demand rule: `import 'dayjs/locale/fr'` registers
French, and nothing you do not import is bundled. The tree of `dayjs/plugin/*` and
`dayjs/locale/*` entry points is what keeps the default import at ~2 kB.

## Production Notes

- **Timezones are not free and not fully self-contained.** The `timezone` plugin
  depends on the host `Intl` time-zone database. In minimal environments (some
  Node builds compiled without full ICU, older React Native/Hermes setups) named
  zones can be missing or wrong. It is also measurably slower than offset math, so
  avoid it in hot loops.
- **Non-ISO parsing silently misbehaves without `customParseFormat`.** Passing a
  string the core cannot recognize can yield an `Invalid Date` (or a lenient guess)
  rather than an error. Always check `.isValid()` on user-supplied input, and load
  the plugin for anything that is not ISO 8601.
- **Format tokens are Moment's, not CLDR's.** `YYYY`, `DD`, `dddd` follow Moment
  convention; teams that also use `Intl.DateTimeFormat` or other libraries should
  not assume token compatibility.
- **Immutability catches Moment migrators off guard.** `d.add(1, 'day')` does not
  change `d`; code ported from Moment that relied on in-place mutation will appear
  to "do nothing." This is a correctness improvement but a real migration gotcha.
- **The API is Moment-compatible, not Moment-complete.** Some Moment behaviors exist
  only via plugins and a few not at all, so "same API" migrations still need an
  audit of edge features (durations, business-day math, strict parsing modes).
- **Still 1.x after years.** Day.js has stayed on a single 1.x line for its entire
  life; a 2.0 has been discussed but not shipped, so do not architect around an
  imminent major release[^2].

## When to Use / When Not

**Use when:**
- You are migrating off Moment.js and want minimal code churn with a smaller bundle.
- Bundle size matters and you only need parse / format / manipulate / basic i18n.
- You want immutable date objects with a familiar chainable API.

**Avoid when:**
- You need first-class, self-contained timezone and calendar correctness — prefer
  Luxon or the native Temporal API.
- You want maximal tree-shaking and no wrapper object — a functional library fits
  better.
- Your target runtime lacks a complete `Intl`/ICU build and you rely on named zones.

## Alternatives

- date-fns/date-fns — functional, tree-shakeable helpers over native `Date`; use it
  when you want to import only the functions you call and skip a wrapper object.
- moment/luxon — from a Moment maintainer, built on `Intl`; use it when timezone and
  internationalization correctness matter more than bundle size.
- tc39/proposal-temporal — the standard Temporal API now reaching runtimes; use it
  when your platform supports it and you want to drop the dependency entirely.
- js-joda/js-joda — immutable model ported from Java's `java.time`; use it when you
  want a rigorous, `Date`-independent domain type.
- moment/moment — the ancestor, in maintenance mode; use it only for legacy code you
  are not yet migrating.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2018-04 | Repository created by iamkun[^1]. |
| 1.0.0 | 2018 | First stable release: immutable core with Moment-style API. |
| 1.8.x | 2019 | Plugin and locale ecosystem broadly filled out. |
| 1.10.x | 2020–2021 | Timezone plugin and parsing fixes; adoption accelerates as Moment enters maintenance[^3]. |
| 1.11.x | 2022–2026 | Long-running current line; incremental fixes, still no 2.0[^2]. |

## References

[^1]: iamkun/dayjs repository, created 2018-04-10. https://github.com/iamkun/dayjs
[^2]: Day.js has remained on the 1.x major line; see releases. https://github.com/iamkun/dayjs/releases
[^3]: Moment.js project status ("we now generally consider Moment to be a legacy project in maintenance mode"). https://momentjs.com/docs/#/-project-status/

## Tags

javascript, date, datetime, time, immutable, moment-alternative, i18n, plugin-architecture, bundle-size, frontend, library
