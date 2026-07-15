# sindresorhus/query-string

> Parse and stringify URL query strings, with array-encoding conventions the native URLSearchParams API deliberately leaves undefined.

[GitHub repo](https://github.com/sindresorhus/query-string) ·
[License: MIT](https://github.com/sindresorhus/query-string/blob/main/license)

## Overview

`query-string` is a small JavaScript library for turning a URL query string (`?foo=bar&foo=baz`) into an object and back again. It has existed since 2013[^1] and predates broad browser support for the native `URLSearchParams` API. Its reason to still exist in 2026 is the part of query-string handling that the specification never pinned down: how to encode an *array* of values under one key. There is no standard for `foo=1&foo=2` versus `foo[]=1&foo[]=2` versus `foo=1,2`, and different backends (PHP, Rails, Express, ASP.NET) expect different conventions. query-string's `arrayFormat` option enumerates seven of them and makes parse/stringify symmetric across each.

The library is authored by Sindre Sorhus and shares his house style: ESM-only, tiny dependency surface, no configuration files, latest-two-browser-versions support. The README itself opens by telling you to consider `URLSearchParams` first[^2] — an unusual honesty for a package with millions of weekly downloads, and a correct one. If you only need to read or set flat string parameters, the platform already does that. query-string earns its place when you need array formats, type coercion (`parseNumbers`, `parseBooleans`), URL-vs-query extraction, or deterministic key sorting.

The defining tension: it is a mature, near-frozen utility whose core job the browser now does natively, kept relevant by a long tail of encoding conventions and ergonomic helpers. New features are rare; the value is in the accumulated edge-case handling.

## Getting Started

```sh
npm install query-string
```

Note the hyphen. The unhyphenated `querystring` on npm is a deprecated, unrelated package[^2].

```js
import queryString from 'query-string';

// parse — leading ? or # is stripped, so location.search/hash work directly
const parsed = queryString.parse('?foo=bar&count=42', {parseNumbers: true});
//=> {foo: 'bar', count: 42}

// stringify — keys are sorted by default for stable output
queryString.stringify({foo: 'unicorn', ilike: 'pizza'});
//=> 'foo=unicorn&ilike=pizza'

// arrays: pick the convention your backend expects
queryString.stringify({ids: [1, 2, 3]}, {arrayFormat: 'bracket'});
//=> 'ids[]=1&ids[]=2&ids[]=3'
queryString.parse('ids[]=1&ids[]=2&ids[]=3', {arrayFormat: 'bracket'});
//=> {ids: ['1', '2', '3']}
```

## Architecture / How It Works

The package is a single module exposing a handful of pure functions: `parse`, `stringify`, `extract`, `parseUrl`, `stringifyUrl`, `pick`, and `exclude`. There is no class, no state, no I/O.

`parse` splits the string on `&`, then splits each pair on the *first* `=` only (so values may contain `=`), decodes with the `decode-uri-component` dependency[^3] (which tolerates malformed percent-encoding that `decodeURIComponent` would throw on), and assembles the result into an object created with `Object.create(null)`. That null prototype is deliberate: it means a query key like `__proto__` or `constructor` cannot pollute the object prototype — a meaningful hardening detail for anything that parses untrusted URLs.

The `arrayFormat` machinery is the substantive part. Each format is a matched pair of a key-parser and a key-formatter:

- `none` (default) — repeated keys collapse into an array (`foo=1&foo=2`).
- `bracket` — `foo[]=1&foo[]=2`.
- `index` — `foo[0]=1&foo[1]=2`, preserving sparse positions.
- `comma` — `foo=1,2,3`, compact but lossy (empty and null elements are indistinguishable after a round trip).
- `separator` — like comma but a configurable `arrayFormatSeparator`.
- `bracket-separator` — brackets mark which keys are arrays, separator joins elements; the only format that distinguishes an empty array (`foo[]`) from a one-empty-string array (`foo[]=`).
- `colon-list-separator` — `foo:list=one&foo:list=two`.

Type coercion (`parseNumbers`, `parseBooleans`) runs after array assembly. The `types` option (a per-key schema of `'number' | 'string' | 'boolean' | 'number[]' | 'string[]' | Function`) overrides the blanket options and is the escape hatch for the classic footgun where an ID like `01234` or a phone number `+380...` gets silently mangled by `parseNumbers`. On stringify, a `replacer` callback (mirroring `JSON.stringify`'s) lets you serialize `Date` and other non-primitives.

## Production Notes

**ESM-only since v8.** Version 8 (2023) dropped CommonJS entirely[^4]. `require('query-string')` throws `ERR_REQUIRE_ESM` on that line of major versions. In a CommonJS codebase you must either use dynamic `import()`, migrate to ESM, or pin to the v7 line. This single change is the most common reason teams are stuck on older query-string, and the most common surprise during dependency upgrades. Jest and other CJS-based test setups frequently need `transformIgnorePatterns` adjustments to handle it.

**`URLSearchParams` covers the flat case.** Before adding this dependency, check whether you actually need array formats or type coercion. `new URLSearchParams(location.search)` and `.toString()` ship in every current runtime with zero install cost. query-string's own docs recommend it for simple use.

**`comma` array format is lossy.** `stringify({foo: [1, null, '']}, {arrayFormat: 'comma'})` yields `foo=1,,` and parsing that back cannot recover the `null`/`''` distinction or the number types. If round-trip fidelity matters, use `bracket-separator` or `index` instead. This is documented but easy to miss.

**Sorting is on by default.** `stringify` sorts keys alphabetically unless you pass `sort: false` or a comparator. This gives stable, cache-friendly URLs but will reorder parameters — surprising if you expected insertion order or if a downstream system is order-sensitive (e.g. signed-request canonicalization).

**`+` decodes to space.** Following form-encoding convention, `+` in a value parses as a space. A literal plus sign must be percent-encoded (`%2B`); this is a recurring bug report, not a defect[^5].

**Nesting is unsupported by design.** There is no `foo[bar]=1` deep-object parsing. The maintainer's position is that nesting is unspecified and implementation-divergent; the recommended pattern is to JSON-stringify the nested value into a single parameter. If you need PHP/Rails-style deep nesting on parse, use `qs` instead.

## When to Use / When Not

**Use when:**
- You must interoperate with a backend that expects a specific array encoding (bracket, index, comma, etc.).
- You want type coercion, per-key type schemas, or a `replacer` for serializing dates/custom types.
- You need `parseUrl`/`stringifyUrl`/`pick`/`exclude` helpers to manipulate whole URLs, not just the query portion.
- You want deterministic, sorted, cache-stable query output.

**Avoid when:**
- You only read/write flat string parameters — use native `URLSearchParams`.
- Your codebase is CommonJS and cannot adopt ESM or dynamic import (unless you pin to v7).
- You need nested-object query strings — use `qs`.
- You want the fastest possible parse in a hot path — the native API is generally faster and allocation-lighter for the flat case.

## Alternatives

- ljharb/qs — the standard when you need nested objects and PHP/Rails-style bracket depth; larger and slower, still CommonJS-friendly.
- whatwg/url (`URLSearchParams`) — native, zero-dependency; use when you only need flat key/value handling.
- unjs/ufo — URL utilities for the modern JS/edge ecosystem; use when you want a broader URL toolkit rather than just query parsing.
- sindresorhus/query-string v7 — pin to the previous major when you cannot move off CommonJS but want the same API.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-11-13 | First published; predates broad `URLSearchParams` support[^1]. |
| 6.0 | 2018 | Major rework of `arrayFormat` handling and options surface. |
| 7.0 | 2021 | Added `types` schema, `pick`/`exclude` helpers, further array formats. |
| 8.0 | 2023 | ESM-only; CommonJS support removed[^4]. |
| 9.0 | 2024 | Latest major line; ESM-only, current browser/runtime targets. |

## References

[^1]: Repository created 2013-11-13. https://github.com/sindresorhus/query-string
[^2]: query-string README — install warning (hyphen; deprecated `querystring`) and `URLSearchParams` recommendation. https://github.com/sindresorhus/query-string#readme
[^3]: `decode-uri-component` — tolerant percent-decoding dependency used by `parse`. https://github.com/SamVerschueren/decode-uri-component
[^4]: Sindre Sorhus on ESM-only package publishing (rationale applied across his libraries, incl. query-string v8). https://gist.github.com/sindresorhus/a39789f98801d908bbc7ff3ecc99d99c
[^5]: query-string issue on `+` being parsed as a space. https://github.com/sindresorhus/query-string/issues/305

## Tags

javascript, query-string, url, urlsearchparams, parsing, serialization, esm, npm-package, browser, nodejs
