# validatorjs/validator.js

> A dependency-free library of string validators and sanitizers — pure `isX(str, options)` predicates, not a schema validator.

[GitHub repo](https://github.com/validatorjs/validator.js) ·
[npm: validator](https://www.npmjs.com/package/validator) ·
[License: MIT](https://github.com/validatorjs/validator.js/blob/master/LICENSE)

## Overview

validator.js is a collection of standalone functions that check whether a string
matches some format (`isEmail`, `isURL`, `isUUID`, `isCreditCard`, `isIBAN`, and
~100 others) or sanitize one (`trim`, `escape`, `normalizeEmail`, `blacklist`).
It descends from Chris O'Hara's original `node-validator` (2010), which used a
chaining API (`check(str).isEmail()`). The 1.0 rewrite dropped chaining in favor
of plain functions that take a string and return a boolean or a transformed
string[^1]. That decision defines the library: it is a bag of small, composable,
side-effect-free predicates with zero runtime dependencies.

The scope is deliberately narrow and worth internalizing before adopting it:
**this library validates strings only.** Every entry point runs an `assertString`
guard and throws on non-string input; the docs tell you to coerce with
`input + ''` if unsure. It does not validate object shapes, does not infer
TypeScript types, and does not produce structured error messages. It answers one
question per call — "is this string a well-formed X?" — and nothing more.

With ~24k stars and tens of millions of weekly npm downloads, it is one of the
most-depended-on validation packages in the JavaScript ecosystem, pulled in
transitively by a large share of Node projects. It remains actively maintained:
the default branch saw commits within days of this writing, though the ~460 open
issues reflect the long tail of locale requests and format edge cases that a
format-matching library inevitably accumulates.

## Getting Started

```sh
npm i validator
```

```javascript
// CommonJS — whole library
const validator = require('validator');

validator.isEmail('foo@bar.com');            // true
validator.isURL('https://example.com');       // true
validator.trim('  hi  ');                      // 'hi'
validator.normalizeEmail('Foo.Bar@Gmail.com'); // 'foobar@gmail.com'
```

```javascript
// ES — import only what you use (smaller browser bundles)
import isEmail from 'validator/lib/isEmail';
import isUUID from 'validator/es/lib/isUUID'; // tree-shakeable build

isEmail('foo@bar.com', { host_blacklist: ['spam.com'] }); // true
```

## Architecture / How It Works

Each validator and sanitizer lives in its own file under `src/`, exports a single
function, and is re-exported from an index that assembles the `validator` object.
The public shapes are consistent: validators are `isX(str[, options]) => boolean`;
sanitizers are `transformX(str[, options]) => string`. Almost every function opens
with `assertString(str)`, so passing a number, `null`, or `undefined` throws
rather than silently coercing.

Internally the library is **regex-plus-options**. Most format checks are one or
more regular expressions guarded by option handling and, for some validators, a
checksum step (Luhn for `isCreditCard`, mod-97 for `isIBAN`, ISO check digits for
`isISBN`/`isISSN`). Locale-sensitive validators — `isAlpha`, `isAlphanumeric`,
`isMobilePhone`, `isPostalCode`, `isFloat`, `isDecimal` — ship large per-locale
tables of character classes and patterns; these tables are the bulk of the
library's size and the source of most feature requests.

The build compiles the ES-module `src/` with Babel into three distributables:
`lib/` (CommonJS, for `require('validator/lib/isEmail')`), `es/` (ES modules, for
tree-shaking bundlers), and a bundled `validator.min.js` UMD file for a
`<script>` tag or CDN. There are **no runtime dependencies**, which is a large
part of the library's appeal: adding it does not expand your dependency tree.

The design consequence to understand: because validation is regex-driven, the
library inherits regex's limits. Email is the canonical example — no regex fully
implements RFC 5322, so `isEmail` makes pragmatic tradeoffs (configurable via
options like `allow_display_name`, `require_tld`, `domain_specific_validation`)
and will both reject some deliverable addresses and accept some undeliverable
ones. `isEmail` is a syntax check, not a proof that mail will arrive.

## Production Notes

**ReDoS is the headline risk.** validator.js has shipped multiple regular-expression
denial-of-service advisories over its history (across validators such as `isEmail`,
`isFQDN`, `isSlug`, `isBase64`, and `rtrim`), where a crafted input triggers
catastrophic backtracking[^2]. Practical mitigations: keep the package updated (do
not pin to an old major indefinitely), cap input length before validating
untrusted strings, and treat validators applied to attacker-controlled input on a
hot path as a security surface, not a formality.

**It is not a schema validator.** validator.js checks one string at a time. For
validating request bodies, config objects, or form payloads you still need a
schema layer on top; validator.js functions are the leaf checks inside it. Teams
that reach for it expecting object validation end up hand-rolling the outer loop.

**Bundle size in the browser.** Importing the whole `validator` object pulls in
every locale table. For client bundles, import individual functions
(`validator/lib/isEmail`) or use the tree-shakeable `validator/es/` build so the
bundler drops unused validators and locale data.

**Types are external.** TypeScript definitions live in `@types/validator` on
DefinitelyTyped, not in the package. The types can lag behind newly added
validators or new option keys, so a validator can exist at runtime before its
type does.

**Behavior drifts across versions.** Option defaults and format coverage change
between releases (email option defaults and locale additions are common examples).
Upgrades are usually smooth, but validators are format opinions, and opinions
shift — pin a version and re-run your own fixtures after bumping, especially for
`isEmail`, `isURL`, and `isMobilePhone`.

## When to Use / When Not

**Use when:**
- You need to validate or sanitize individual strings server-side (email, URL,
  UUID, credit card, IBAN, ISO codes, crypto addresses) with wide format coverage.
- You want zero runtime dependencies and small, composable functions.
- You need sanitizers (`escape`, `trim`, `normalizeEmail`, `blacklist`) alongside
  validators.

**Avoid when:**
- You need to validate object/request shapes or want TypeScript type inference —
  that is a schema library's job, not this one's.
- You are validating untrusted input at scale without budget to review ReDoS
  exposure and cap input lengths.
- You want rich, structured validation errors for form UIs — these functions
  return only booleans.

## Alternatives

- colinhacks/zod — TypeScript-first schema validation with static type inference; use instead when validating structured data and you want types derived from the schema.
- hapijs/joi — mature object-schema validation for server-side payloads; use when you need declarative schemas and detailed error objects rather than leaf string checks.
- jquense/yup — schema validation with a fluent API, popular in React form stacks; use when validating form objects with per-field messages.
- ajv-validator/ajv — JSON Schema validation; use when your contracts are already expressed as JSON Schema (APIs, config).
- Note: these operate at the schema layer; validator.js operates at the string layer, and the two are frequently combined rather than chosen between.

## History

| Version | Date | Notes |
|---------|------|-------|
| node-validator | 2010 | Original by Chris O'Hara; chaining API (`check(str).isEmail()`)[^1]. |
| 1.0 | 2013 | Rewrite to standalone, string-only functions; chaining removed[^1]. |
| 7.x | 2017 | Continued expansion of validators and locale coverage[^3]. |
| 10.x | 2018 | Broader locale tables; more crypto/ISO validators[^3]. |
| 13.0 | 2020 | Current major line; ongoing validator and locale additions[^3]. |

## References

[^1]: validator.js README and project history, validatorjs/validator.js. https://github.com/validatorjs/validator.js
[^2]: npm advisories for the `validator` package (ReDoS class). https://www.npmjs.com/advisories?search=validator
[^3]: `validator` release history on npm. https://www.npmjs.com/package/validator?activeTab=versions

## Tags

javascript, validation, sanitization, string, regex, npm, node, input-validation, email, security, zero-dependency
