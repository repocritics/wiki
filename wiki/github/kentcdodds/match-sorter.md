# kentcdodds/match-sorter

> Simple, expected, and deterministic best-match sorting of a JavaScript array — a fixed ranking ladder, not a fuzzy scoring engine.

[GitHub repo](https://github.com/kentcdodds/match-sorter) ·
[npm](https://npm.im/match-sorter) ·
[License: MIT](https://github.com/kentcdodds/match-sorter/blob/main/LICENSE)

## Overview

match-sorter is a small client-side library that filters and sorts an array against a
user-typed string. It was written by Kent C. Dodds and grew out of `genie`, described in
the README as the first library he ever wrote[^1]. The problem it targets is the everyday
"type-to-filter" input: a list of dozens to a few thousand items that should reorder
sensibly as the user types, without the results jumping around in ways that feel random.

Its defining decision is to reject probabilistic fuzzy matching. Instead of a distance
score (as in Fuse.js), every item is bucketed into one of a fixed set of ranked tiers —
case-sensitive equal, equal, starts-with, word-starts-with, contains, acronym, and a
loose in-order "matches" — and ties are broken by a stable alphabetic sort[^2]. The result
is deterministic and explainable: for a given input you can predict the order, and the
order does not "fancily change" mid-keystroke. That predictability is the whole product,
and it is also the ceiling — match-sorter deliberately does not do typo tolerance,
weighted relevance, or semantic ranking.

It is a filtering primitive, not a search engine. There is no index, no persistence, and
no async. Every call is a synchronous pass over the array you hand it, which is why it fits
naturally inside React render logic and controlled inputs but does not scale to large
corpora.

## Getting Started

```bash
npm install match-sorter
```

```javascript
import {matchSorter} from 'match-sorter'

const list = ['hi', 'hey', 'hello', 'sup', 'yo']
matchSorter(list, 'h') // ['hello', 'hey', 'hi']
matchSorter(list, 'z') // []

// object arrays: rank against specific keys
const people = [
  {name: 'Janice', color: 'Green'},
  {name: 'Fred', color: 'Orange'},
  {name: 'George', color: 'Blue'},
]
matchSorter(people, 'g', {keys: ['name', 'color']})
// George (name starts-with) ranks above Janice/Fred (color contains)
```

Keys support dot-notation for nested fields (`'name.first'`), numeric indices
(`'aliases.0.name'`), a `*` wildcard across arrays (`'aliases.*.name.first'`), and
function callbacks (`item => item.name`) for computed or Immutable.js-backed values.

## Architecture / How It Works

The core is a single synchronous function. For each item it computes the highest-ranking
match across all configured keys, drops anything below the threshold, then sorts the
survivors. Ranks are exposed as `matchSorter.rankings` constants, top to bottom:
`CASE_SENSITIVE_EQUAL`, `EQUAL`, `STARTS_WITH`, `WORD_STARTS_WITH`, `CONTAINS`, `ACRONYM`,
`MATCHES` (the default threshold), and `NO_MATCH`. The `SIMPLE_MATCH`/`MATCHES` tier checks
whether the item's characters appear in the same order as the input, and rewards closer
spans so shorter items outrank longer ones on an equal in-order match[^2].

Configuration is per-call via the options object. `keys` accepts plain strings, callbacks,
or objects that carry per-key `threshold`, `minRanking`, and `maxRanking` — so a given
field can be forced to demand at least `STARTS_WITH`, or capped so an exact hit on a low-value
field cannot outrank a partial hit on a more important one. `baseSort` controls tie-breaking
(default `String.localeCompare` on the ranked value, giving stable alphabetic order), and
`sorter` replaces the whole post-match sort — passing `rankedItems => rankedItems` disables
sorting entirely while keeping the filter.

`matchSorterWithRankInfo` is the same computation but returns `{item, rank, keyIndex,
keyThreshold, index, rankedValue}` rows instead of bare items, for callers that need the
scoring metadata (e.g. to render why something matched).

The dependency footprint is two packages: `@babel/runtime` (helper sharing) and
`remove-accents` for diacritic folding. By default match-sorter strips diacritics before
comparing, so `café` matches `cafe`; `keepDiacritics: true` turns that off. The library is
written in TypeScript and ships types.

## Production Notes

- **It is O(n) per keystroke.** Every call rescans the entire array; there is no index or
  memoization built in. For a few thousand rows in a controlled input this is fine, but
  debounce the input and/or memoize results for large lists — re-running on every render is
  the most common performance mistake.
- **Multi-word queries do not span keys by default.** A search of "two words" is treated as
  one string and must match a single field in full. The documented workaround is to split
  on spaces and chain calls with `reduceRight`, which lets each word match a different
  column[^3]. This is a recipe, not built-in behavior — expect to write it yourself.
- **Non-space word separators need a callback.** Word-boundary ranks (`WORD_STARTS_WITH`)
  assume spaces. For `snake_case`, `camelCase`, or `kebab-case` data you must normalize via a
  key callback (e.g. `item => item.name.replace(/_/g, ' ')`) or those bonuses won't fire.
- **Diacritic stripping is on by default.** Good for UX, but it means `keepDiacritics:false`
  data will match across accent variants whether you intended it or not; be deliberate when
  the distinction is meaningful (names, non-Latin scripts).
- **`keys[*].threshold` vs top-level `threshold`.** Mixing per-key and global thresholds is a
  frequent source of "why is this item showing / missing" confusion. The per-key value wins
  for that key; the global one applies to keys that don't set their own.
- **Not a search backend.** No fuzzy typo correction, no stemming, no relevance tuning
  beyond the fixed ladder. If users expect Google-style forgiveness, match-sorter will feel
  rigid — that rigidity is the design, not a bug.

## When to Use / When Not

**Use when:**
- You have a bounded in-memory list (dozens to low thousands) and a type-to-filter input.
- You want predictable, explainable ordering that won't reshuffle unexpectedly as users type.
- You want a tiny dependency with a zero-config default and no index to build or maintain.

**Avoid when:**
- You need typo tolerance, weighted relevance, or fuzzy scoring — reach for Fuse.js.
- Your dataset is large or server-side — use a real search index (Lucene/Elastic, or a
  client index like FlexSearch/Lunr) rather than an O(n) rescan.
- You need multi-field, multi-word relevance out of the box — match-sorter makes you assemble it.

## Alternatives

- krisk/Fuse — fuzzy scoring with typo tolerance and weighted keys; use when "close enough" matches matter more than deterministic order.
- nextapps-de/flexsearch — in-memory full-text index; use when the list is large enough that per-keystroke rescans hurt.
- olivernn/lunr.js — small client-side inverted index with relevance scoring; use when you want real search semantics offline.
- farzher/fuzzysort — fast fuzzy substring ranking optimized for large lists; use when speed on big arrays beats explainability.
- kbrsh/leven / word-distance libs — raw edit-distance primitives; use when you want to build your own ranking rather than adopt a ladder.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2016-08-25 | Extracted from Kent C. Dodds' `genie`; first published[^1]. |
| 3.0.0 | 2019-04-17 | Major API iteration. |
| 4.0.0 | 2019-07-19 | Breaking changes to matching/options. |
| 5.0.0 | 2020-10-12 | Major release. |
| 6.0.0 | 2020-11-26 | Long-lived 6.x line (nested keys, wildcard, callbacks matured). |
| 7.0.0 | 2024-10-17 | Major bump after the 6.x era. |
| 8.0.0 | 2024-11-04 | Current major line. |
| 8.3.0 | 2026-04-15 | Latest release as of mid-2026[^4]. |

Around 4.1k stars and 143 forks; maintenance is low-volume but current, with the most
recent push in May 2026 and a small open-issue count — a stable, essentially "done" utility
rather than an actively expanding project.

## References

[^1]: match-sorter README, "Inspiration" — extracted from the `genie` library. https://github.com/kentcdodds/match-sorter#inspiration
[^2]: match-sorter README, ranking criteria and `threshold` section. https://github.com/kentcdodds/match-sorter#threshold-number
[^3]: match-sorter README, "Match many words across multiple fields (table filtering)". https://github.com/kentcdodds/match-sorter#recipes
[^4]: match-sorter GitHub releases. https://github.com/kentcdodds/match-sorter/releases

## Tags

javascript, typescript, fuzzy-search, filtering, sorting, array, search, frontend, ui, utility, deterministic
