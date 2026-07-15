# mathiasbynens/he

> A spec-exhaustive HTML entity encoder/decoder for JavaScript that gets the ugly edge cases right.

[GitHub repo](https://github.com/mathiasbynens/he) ·
[Official website](https://mths.be/he) ·
[License: MIT](https://github.com/mathiasbynens/he/blob/master/LICENSE-MIT.txt)

## Overview

_he_ (short for "HTML entities") is a small, zero-dependency JavaScript library
that encodes text to HTML character references and decodes them back[^1]. It was
written by Mathias Bynens — the author of several reference-grade string and
Unicode utilities — and first published in 2013. Its selling point is not speed
or size but correctness: it implements the WHATWG HTML tokenizer's character
reference rules faithfully, including the cases most hand-rolled solutions get
wrong.

The defining detail is that _he_ treats the HTML spec as ground truth rather than
approximating it. It knows the full standardized named-reference table (`&copy;`,
`&ne;`, and ~2,000 others), handles ambiguous ampersands (`&ampbar` vs
`&amp;bar`), distinguishes decoding in a text context from decoding inside an
attribute value (the spec has different rules for each), and round-trips astral
(non-BMP) symbols such as `𝌆` (U+1D306) correctly instead of mangling surrogate
pairs[^2]. Many one-line regex "entity decoders" fail on exactly these inputs.

The tension is scope. _he_ is deliberately just an entity codec, not an HTML
sanitizer or parser. It will faithfully decode `&lt;script&gt;` back into
`<script>`; using it to "clean" untrusted HTML is a security mistake. It is also
effectively finished software — the last commit landed in 2021 and there is no
active roadmap. For its narrow job that stability is a feature; for anyone
expecting ongoing development it can read as abandonment.

## Getting Started

```bash
npm install he
```

```js
const he = require('he');

// Encode: non-ASCII and unsafe chars become references
he.encode('foo © bar ≠ baz 𝌆 qux');
// → 'foo &#xA9; bar &#x2260; baz &#x1D306; qux'

he.encode('foo © bar', { useNamedReferences: true });
// → 'foo &copy; bar'

// Decode: named + numeric references back to text
he.decode('foo &copy; bar &ne; baz &#x1D306; qux');
// → 'foo © bar ≠ baz 𝌆 qux'

// escape(): the minimal, safe-for-markup subset only
he.escape('<img src=\'x\' onerror="prompt(1)">');
// → '&lt;img src=&#x27;x&#x27; onerror=&quot;prompt(1)&quot;&gt;'
```

A CLI ships with the package (`npm install -g he`, then `he --encode` /
`he --decode`, stdin/stdout friendly).

## Architecture / How It Works

The published library is a single generated file (`he.js`, UMD) plus a bundled
`data/decode-map.json` derived from the HTML spec's named-character-reference
table. The maintainer regenerates that data from the WHATWG source via a Grunt
build step, so the reference set tracks the standard rather than being
hand-curated[^2].

Four public functions cover the surface:

- **`encode(text, options)`** — escapes anything outside printable ASCII plus the
  markup-unsafe set (`& < > " ' ` `` ` ``). Options tune the output form:
  `useNamedReferences` (emit `&copy;` instead of `&#xA9;`), `decimal`
  (`&#169;` instead of hex), `encodeEverything`, `allowUnsafeSymbols`, and
  `strict` (throw on code points the spec forbids rather than passing them
  through).
- **`decode(html, options)`** — implements the spec's "tokenizing character
  references" state machine. The `isAttributeValue` flag switches to the
  attribute-value parsing rules, where a trailing-alphanumeric ambiguous
  ampersand is left untouched (`foo&ampbar` → `foo&ampbar`) instead of decoded.
  `strict` throws on parse errors, which is what lets _he_ double as a validator.
- **`escape(text)`** — the fast path: escapes only the six markup-significant
  characters, no table lookups. This is the function most callers actually want.
- **`unescape`** — an alias for `decode`.

Global defaults live on `he.encode.options` / `he.decode.options`, so callers can
set behavior once instead of passing an options object every call. Everything is
synchronous and pure; there is no I/O and no state beyond those option objects.

## Production Notes

**It is a codec, not a sanitizer.** The single most important operational fact:
decoding untrusted input with _he_ and then inserting the result into the DOM
re-introduces any markup the input encoded. For XSS defense you want output
escaping (`he.escape`, or your template engine's autoescaping) or a real
sanitizer like DOMPurify — never `he.decode`.

**Module format is dated.** _he_ predates ES modules and ships UMD/CommonJS only.
It works everywhere (Node, bundlers, `<script>`, AMD) but there is no native ESM
entry point, so tree-shaking does not apply and modern ESM-only toolchains import
it through interop. The whole library is small enough (single file, one JSON
table) that this rarely matters for bundle size in practice.

**Named references are larger and less compatible.** `useNamedReferences: true`
produces prettier output but the note in the docs stands: very old browsers do
not recognize every named entity, so the default (hex numeric escapes) is the
safe interoperable choice. Leave it off unless you control the consumer.

**Performance is adequate, not exceptional.** `decode` walks the spec state
machine and consults the reference table; for high-throughput HTML processing
(parsing megabytes per request) the `entities` package is measurably faster and
integrates with the htmlparser2/cheerio stack. For typical encode/escape of user
strings the difference is irrelevant.

**Maintenance is dormant.** No releases since 1.x-era and no commits after 2021.
Because the HTML named-reference table is itself frozen and the code has no
dependencies to rot, "unmaintained" here means "stable," not "vulnerable" — but
do not expect bug fixes or new options. Pin it and move on.

## When to Use / When Not

**Use when:**
- You need correct decoding of arbitrary HTML entities, including named,
  decimal, hex, ambiguous-ampersand, and astral cases.
- You need context-aware decoding (text vs attribute value) matching browser
  behavior.
- You want `strict` mode to reject malformed references for a parser/validator.
- You want a dependency-free, everywhere-runnable codec you can pin and forget.

**Avoid when:**
- You are trying to sanitize untrusted HTML — use DOMPurify; _he_ is the wrong
  layer.
- You only ever escape the six markup characters and want zero dependency — a
  four-line replace or your framework's autoescaping suffices.
- You are in a hot parsing path and need maximum decode throughput — prefer
  `entities`.
- You require a native ESM/TypeScript-first package with active maintenance.

## Alternatives

- fb55/entities — faster encode/decode, native ESM + TypeScript types, and the codec used inside htmlparser2/cheerio; use it when throughput or module hygiene matters.
- mdevils/html-entities — modern TypeScript/ESM package with configurable entity levels; use it when you want an actively maintained, tree-shakeable dependency.
- cure53/DOMPurify — an actual HTML sanitizer; use it when the goal is XSS-safe rendering of untrusted markup, not entity round-tripping.
- substack/node-ent — minimal, older escaper; use it only for trivial ASCII escaping where edge cases are irrelevant.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-06-25 | First commit; entity encode/decode + CLI[^1]. |
| 1.x | 2015–2018 | Astral handling, `strict`, `isAttributeValue`, decimal/named options settle in. |
| — | 2021-12-29 | Last commit to `master`; project effectively frozen[^3]. |

## References

[^1]: _he_ README and API documentation. https://github.com/mathiasbynens/he#readme
[^2]: WHATWG HTML Standard, "Named character references" and "Tokenizing character references." https://html.spec.whatwg.org/multipage/syntax.html#named-character-references
[^3]: Repository metadata (last push 2021-12-29) via GitHub API — mathiasbynens/he.

## Tags

javascript, html-entities, encoder, decoder, unicode, escaping, html, whatwg, cli, zero-dependency
