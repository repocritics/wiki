# simov/slugify

> A dependency-free string-to-slug converter built around a hand-maintained Unicode transliteration table.

[GitHub repo](https://github.com/simov/slugify) ·
[npm](https://www.npmjs.com/package/slugify) ·
[License: MIT](https://github.com/simov/slugify/blob/master/LICENSE)

## Overview

`slugify` turns arbitrary strings into URL-safe slugs: `slugify('Héllo Wörld')` yields `Hello-World`. It is one of the oldest and most-installed libraries in this niche, created in 2013[^1] as a vanilla-JavaScript port of dodo/node-slug. Its scope is deliberately narrow — replace whitespace with a separator, transliterate accented and foreign characters to their closest ASCII equivalent, and optionally lowercase or strip the rest.

The library's real substance is not its code (a single small `index.js`) but its data: a hand-curated `charmap.json` mapping thousands of Unicode code points to ASCII strings, plus a `locales.json` that overrides individual mappings per language[^2]. A character's transliteration is therefore a policy decision baked into a JSON file, not an algorithm. This is the project's defining tradeoff: coverage is broad but finite, additions arrive by pull request, and any script whose glyphs are not in the map is silently dropped rather than romanized.

The package is stable rather than actively developed. At ~1,700 stars it is widely depended upon, but commit activity is sparse and the API has not meaningfully changed in years — most churn is community-contributed character-map entries.

## Getting Started

```bash
npm install slugify
```

```js
const slugify = require('slugify')

slugify('some string')            // 'some-string'
slugify('some string', '_')       // 'some_string'  (separator shorthand)

slugify('Héllo Wörld', {
  lower: true,                    // lowercase the result
  strict: true,                   // drop anything that isn't [a-z0-9-]
  trim: true                      // trim leading/trailing separators
})                                // 'hello-world'
```

Full option set: `replacement` (separator, default `-`), `remove` (a global-flag character-class RegExp of chars to delete), `lower`, `strict`, `locale` (ISO 639-1 code selecting a `locales.json` override), and `trim`[^2]. The module also exposes itself as `window.slugify` in browsers and supports AMD/CommonJS loaders.

## Architecture / How It Works

There is essentially one function and two data files. At require time `index.js` embeds the character map (the test suite regenerates the inlined map from `config/charmap.json` on `npm test`[^3]). Slugification is a single left-to-right pass: for each character, look up a replacement in the active map, fall back to the raw character, then apply whitespace-to-`replacement` substitution and the `strict`/`remove`/`lower`/`trim` post-processing.

Transliteration is table-driven, not rule-driven. There is no ICU, no `String.prototype.normalize` NFD decomposition, and no phonetic engine. `é` maps to `e` because someone added that pair to `charmap.json`; `☢` produces nothing because no one did. `locales.json` layers language-specific overrides on top — for example the same Cyrillic or German character can romanize differently under a `locale: 'de'` vs `locale: 'uk'` selection.

The `extend` method lets callers add or override mappings:

```js
slugify.extend({ '☢': 'radioactive' })
slugify('unicode ♥ is ☢')   // 'unicode-love-is-radioactive'
```

The critical detail is that `extend` mutates the module-level map for the **entire process**, not a per-call instance. There is no factory or options-scoped map; every consumer of the same required module shares one mutable table. To get a clean map you must bust the require cache (`delete require.cache[require.resolve('slugify')]`) and re-require[^2].

## Production Notes

- **`extend` is global, shared, and permanent.** In a long-running server or a bundle where two libraries both depend on `slugify`, one caller's `extend` silently changes slugs everywhere. Treat map extension as process-wide configuration set once at startup, never per-request.
- **Coverage is a data table, so gaps are silent.** CJK, most Indic scripts, emoji, and many symbols are not in the default map and vanish from the output. Two distinct inputs can collapse to the same slug (or to an empty string), which breaks uniqueness assumptions if you use slugs as identifiers without a suffix or collision check. If you need real CJK-to-pinyin/romaji transliteration, this is the wrong tool.
- **`strict` and `remove` overlap confusingly.** `strict: true` strips everything outside `[A-Za-z0-9]` and the replacement char; `remove` deletes chars matching a supplied RegExp *before* separators are applied. The docs warn that `remove` must be a global-flag character class and nothing else — a non-global or non-class RegExp misbehaves quietly[^2].
- **No lowercasing by default.** `lower` defaults to `false`, so out of the box you get mixed-case slugs — a frequent surprise for people who expect URL slugs to be lowercase.
- **Old-browser support was dropped.** The current line is ES2015 vanilla JS with no transpile step; the last release targeting older browsers is v1.4.7[^4]. Ship it through your own bundler/transpiler if you must support legacy targets.
- **Name collision with `slug`.** The similarly-named `slug` npm package is a different project with broader Unicode handling; imports and issue threads are easy to mix up.

## When to Use / When Not

**Use when:**
- You need Latin/European accented text turned into clean ASCII slugs with zero dependencies and a tiny footprint.
- You want a small, stable, well-understood function for titles, filenames, or URL paths.
- You're fine curating a few `extend` entries for the handful of symbols you care about.

**Avoid when:**
- Your inputs are heavily CJK, Indic, Arabic, or emoji and you need faithful romanization — the map will drop them.
- You need per-instance or per-request character maps — the global mutable `extend` model doesn't fit.
- You require guaranteed-unique slugs from arbitrary Unicode without your own collision handling.

## Alternatives

- sindresorhus/slugify — more opinionated URL slugs; splits camelCase, handles German umlauts and separators smartly. Use when you want batteries-included defaults over a raw transliteration table.
- Trott/slug — the `slug` package this project descends from; broader Unicode coverage and a mode-based charmap. Use when default-map gaps bite you.
- pid/speakingurl — heavier and highly configurable with many language modes. Use when you need fine-grained per-language control.
- lovell/limax — wraps transliteration for CJK/Cyrillic via unidecode-style tables. Use when you need real Chinese/Japanese romanization.
- lodash/lodash `kebabCase` — trivial ASCII-only casing. Use when input is already ASCII and you just need word-joining.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-02-05 | Repo created; vanilla-JS port of node-slug[^1]. |
| 1.4.7 | 2021-02-21 | Last release supporting older browsers[^4]. |
| 1.5.0 | 2021-03-20 | Charmap/locale expansion. |
| 1.6.0 | 2021-07-15 | Adds the `trim` option; ES2015-only baseline. |
| 1.6.8 | 2026-03-13 | Latest tag; ongoing charmap contributions[^5]. |

## References

[^1]: simov/slugify repository, created 2013-02-05. https://github.com/simov/slugify
[^2]: slugify README — Options, Locales, and Extend sections. https://github.com/simov/slugify#readme
[^3]: slugify Contribute section — tests regenerate the inlined charmap and sort `charmap.json`. https://github.com/simov/slugify#contribute
[^4]: slugify v1.4.7 release — last version targeting older browsers. https://github.com/simov/slugify/releases/tag/v1.4.7
[^5]: slugify tags (latest v1.6.8, 2026-03-13). https://github.com/simov/slugify/tags

## Tags

javascript, nodejs, slug, slugify, transliteration, url-slug, string-utils, unicode, zero-dependency, browser
