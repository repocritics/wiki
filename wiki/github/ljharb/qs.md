# ljharb/qs

> A querystring parser and serializer for Node and the browser, with nested-object and array support that native `URLSearchParams` does not provide.

[GitHub repo](https://github.com/ljharb/qs) ·
[npm](https://www.npmjs.com/package/qs) ·
[License: BSD-3-Clause](https://github.com/ljharb/qs/blob/main/LICENSE.md)

## Overview

qs parses and stringifies URL query strings, but its reason to exist is nesting: it maps bracket syntax like `foo[bar][baz]=x` to and from real JavaScript objects and arrays, which the WHATWG `URLSearchParams` standard deliberately does not do. It began life as TJ Holowaychuk's `node-querystring`, the query parser behind early Express/Connect, and is now maintained by Jordan Harband (ljharb)[^1]. It is a CommonJS module with a near-zero dependency footprint, and one of the most-downloaded packages on npm — pulled in transitively by Express (`extended` body parsing), request libraries, and much of the older Node ecosystem.

The defining tension is feature-richness versus safety and speed. Because qs reconstructs arbitrary nested structure from attacker-controllable strings, most of its history is a series of hardening steps — depth limits, parameter limits, prototype-pollution fixes — layered onto an API that must stay backward compatible with a decade of callers. It is more capable than the platform standard and slower than minimal alternatives, and it carries a security surface those alternatives do not have. If you only need flat key/value pairs, qs is more library than the job requires.

## Getting Started

```bash
npm install qs
```

```javascript
const qs = require('qs');

// Nesting is the whole point:
qs.parse('foo[bar]=baz&list[]=a&list[]=b');
// => { foo: { bar: 'baz' }, list: ['a', 'b'] }

qs.stringify({ foo: { bar: 'baz' }, list: ['a', 'b'] });
// => 'foo%5Bbar%5D=baz&list%5B0%5D=a&list%5B1%5D=b'

// Hardened parse for untrusted input:
qs.parse(userInput, {
  depth: 5,
  parameterLimit: 100,
  strictDepth: true,          // throw instead of silently stopping
  throwOnLimitExceeded: true, // throw instead of silently truncating
});
```

## Architecture / How It Works

qs is two small pieces of logic — `lib/parse.js` and `lib/stringify.js` — over shared helpers in `lib/utils.js` and `lib/formats.js`. There is no grammar or state machine; parsing splits on the delimiter, then walks each key's bracket segments and merges the result into an accumulator object. This "merge as you go" model is why mixed notations produce predictable-but-surprising shapes: `a[0]=b&a[b]=c` becomes `{ a: { '0': 'b', b: 'c' } }` because an object and an array key collided and the object won.

Parsing is intentionally lossy toward safety. Arrays are compacted (sparse indices like `a[1]&a[15]` collapse to two elements unless `allowSparse` is set), and a numerically-indexed collection silently converts to a plain object once an index exceeds `arrayLimit` (default 20) — a guard against someone sending `a[999999999]` and forcing a giant allocation. Everything is parsed as a string by design; qs will not coerce `a=15` to a number, and the maintainer has stated this will not change[^2].

Stringify is the inverse and depends on ljharb's `side-channel` package to detect circular references without a global map, throwing on cycles rather than looping forever. Array serialization is governed by `arrayFormat` (`indices`, `brackets`, `repeat`, `comma`), and the four formats do not all round-trip identically — `comma` in particular cannot represent nested objects and needs `commaRoundTrip` to survive single-element arrays. The library ships as CommonJS; ESM consumers use a default import and interop, and there is no separate ESM build.

## Production Notes

- **Prototype pollution is a real, recurring class of bug here.** qs ignores keys that would overwrite `Object.prototype` by default, but `allowPrototypes: true` re-enables that footgun, and a historical gap (CVE-2022-24999, fixed in 6.10.3) let crafted `__proto__` keys pollute prototypes through Express before the qs bump propagated[^3]. For untrusted input prefer `plainObjects: true` (null-prototype results via `{ __proto__: null }`) and never enable `allowPrototypes`.

- **The defaults are DoS mitigations, and the silent ones can bite.** `depth` (5), `parameterLimit` (1000), and `arrayLimit` (20) all default to *silently* stopping/converting rather than erroring. That means truncated data reaches your handler looking valid. Set `strictDepth: true` and `throwOnLimitExceeded: true` on any untrusted path so limit violations surface as catchable errors instead of quietly wrong objects.

- **`arrayLimit` is a representation threshold, not a hard cap.** Exceeding it converts the array to an object but still holds every element — it does not reject or truncate. Likewise `parameterLimit` only bounds `&`-delimited pairs; with `comma: true` a single parameter can still expand into arbitrarily many elements. Bound the raw input at the transport layer (HTTP body-size limit) regardless.

- **`stringify` depth defaults to `Infinity`.** Parse is bounded; stringify is not, on the theory that its input is your own object. If the object's nesting can be influenced by untrusted data, pass a numeric `depth` or a deep enough structure will overflow the call stack instead of throwing a `RangeError`.

- **It is not the fastest option.** The bracket-walking and merge logic make qs meaningfully slower than `fast-querystring` or native `URLSearchParams` for flat parsing. On hot request paths that only need flat pairs, qs is often the wrong default carried in by a framework rather than a deliberate choice.

- **Historical DoS advisory.** Versions before 6.3.2 / 6.2.3 / 6.1.2 / 6.0.4 were vulnerable to a parsing DoS (CVE-2017-1000048)[^4]. Long-lived services on ancient transitive qs pins should audit the resolved version, not the declared range.

## When to Use / When Not

**Use when:**
- You need to round-trip nested objects or arrays through a query string (`filter[status][]=open`).
- You're already in an Express/older-Node stack where qs is the de facto convention.
- You need charset handling, custom encoders/decoders, dot notation, or configurable array formats that the platform standard lacks.

**Avoid when:**
- You only need flat key/value pairs — native `URLSearchParams` is built in, standard, and has no nesting attack surface.
- Parse throughput on untrusted input is a bottleneck — a flat, minimal parser is faster and smaller.
- You want an ESM-first, tree-shakeable dependency — qs is CommonJS-first and feature-monolithic.

## Alternatives

- sindresorhus/query-string — simpler ESM library; flat-first with some array support, no deep object nesting. Use when you want a modern, lighter API and don't need bracket nesting.
- anonrig/fast-querystring — performance-focused flat parser. Use when you parse large volumes of flat query strings and speed matters more than features.
- WHATWG URLSearchParams (built into Node and browsers) — the standard. Use when you only need flat pairs and want zero dependencies and zero attack surface.
- Node's built-in node:querystring — legacy core module, flat only. Use for simple internal parsing where you don't want any dependency; avoid for new nested use cases.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2014-07-25 | Repo created; continuation of TJ Holowaychuk's node-querystring under ljharb[^1]. |
| 6.0.x | 2015 | Modern 6.x line begins; options-object API, depth/array limits. |
| 6.0.4–6.3.2 | 2017 | DoS hardening across the 6.0–6.3 branches (CVE-2017-1000048)[^4]. |
| 6.10.3 | 2022 | Prototype-pollution fix for `__proto__` handling (CVE-2022-24999)[^3]. |
| 6.x (current) | 2024–2026 | Added `strictDepth`, `throwOnLimitExceeded`, `allowEmptyArrays`, `decodeDotInKeys`, `duplicates`; active maintenance continues (last push 2026-07-11). |

## References

[^1]: qs README — "originally created and maintained by TJ Holowaychuk (node-querystring); Lead Maintainer: Jordan Harband." https://github.com/ljharb/qs
[^2]: qs issue #91 — values are always parsed as strings by design. https://github.com/ljharb/qs/issues/91
[^3]: CVE-2022-24999 — qs prototype pollution, fixed in 6.10.3 (impacting Express). https://github.com/advisories/GHSA-hrpp-h998-j3pp
[^4]: CVE-2017-1000048 — qs DoS via crafted input before 6.3.2 / 6.2.3 / 6.1.2 / 6.0.4. https://github.com/advisories/GHSA-gqgv-6jq5-jjj9

## Tags

javascript, nodejs, querystring, url-parsing, serialization, parsing, security, prototype-pollution, commonjs, npm-package
