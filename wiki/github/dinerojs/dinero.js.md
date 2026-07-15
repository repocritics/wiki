# dinerojs/dinero.js

> Immutable, tree-shakeable money math for JavaScript and TypeScript — integer minor units, no floats.

[GitHub repo](https://github.com/dinerojs/dinero.js) ·
[Official website](https://dinerojs.com) ·
[License: MIT](https://github.com/dinerojs/dinero.js/blob/main/LICENSE)

## Overview

Dinero.js exists because `0.1 + 0.2 !== 0.3` and money cannot tolerate that. It models
a monetary value as an integer `amount` in the currency's smallest unit (cents, not
dollars), a `currency` descriptor carrying its exponent/scale, and an explicit `scale`
so sub-cent precision is representable[^1]. All arithmetic stays in integers, sidestepping
IEEE-754 float drift. Created by Sarah Dayan, the repo dates to 2018[^2] and is broadly
depended on (WooCommerce, Cypress fixtures, and many others cite it).

The library has two incompatible lives. **v1** was an object-oriented, chainable API
(`Dinero({ amount: 500 }).add(...)`) shipped as a single module. **v2** is a ground-up
rewrite: standalone pure functions you compose, TypeScript-first with full inference, and
tree-shakeable so you bundle only the operations you import. The two share a name and a
philosophy but no code and no API surface.

The defining fact about this project is timing, not design. v2's first alpha shipped in
July 2021; it then sat in `alpha`/`beta` for nearly five years — including a roughly
three-year dormant stretch after `alpha.14` (Feb 2023) — before `2.0.0` finally went stable
in March 2026[^3]. For most of that window the only way to use the modern API was to pin
`dinero.js@alpha` in production, and much of the ecosystem simply stayed on v1. Evaluate this
repo with that history in mind: the API is good, but "actively maintained" is a recent state,
not a continuous one.

## Getting Started

```sh
npm install dinero.js
```

```js
import { dinero, add, toDecimal } from 'dinero.js';
import { USD } from 'dinero.js/currencies';

const d1 = dinero({ amount: 500, currency: USD });   // $5.00
const d2 = dinero({ amount: 800, currency: USD });   // $8.00

toDecimal(add(d1, d2));   // "13.00"  — amount is in minor units (cents)
```

Formatting is deliberately not built in. Dinero exposes the value; you own the locale layer:

```js
import { toDecimal } from 'dinero.js';

const money = dinero({ amount: 1099, currency: USD });

toDecimal(money, ({ value, currency }) =>
  Number(value).toLocaleString('en-US', { style: 'currency', currency: currency.code })
); // "$10.99"
```

## Architecture / How It Works

A `Dinero` object is a closed, immutable value: `{ amount, currency, scale }` plus internal
metadata. There are no methods on it. Every operation — `add`, `subtract`, `multiply`,
`allocate`, `convert`, `compare`, `toDecimal`, `toSnapshot` — is a free function imported from
the package and returns a **new** object; nothing mutates in place[^1]. That functional shape
is what makes the library tree-shakeable: a bundler drops any operation you never import.

Numeric precision is pluggable through a **calculator**. By default amounts are JS `number`s,
which is fine until totals exceed `Number.MAX_SAFE_INTEGER` (roughly 90 trillion in cents). For
larger values or higher scales you swap in the `bigint` calculator and construct via the
`@dinero.js/core` factory[^4]. The calculator is the seam that lets the same operations run over
different number backends.

**Scale** is the subtle part. `add(a, b)` requires both operands to share a currency *and* a
scale; Dinero will normalize scales internally when it can, but operations that change precision
(division, `allocate`, `transformScale`) can raise the scale, and comparing or combining values
of different scale is where most user bugs originate. Non-decimal and multi-subdivision
currencies (bases other than 10, e.g. historic or crypto units) are supported precisely because
scale and currency exponent are first-class rather than assumed to be "hundredths"[^5].

Currencies are a separate concern. You import currency descriptors from `dinero.js/currencies`
rather than passing strings, so the currency's code, base, and exponent travel with the value
and the type system knows about them.

## Production Notes

- **The alpha era leaves residue.** Tutorials, Stack Overflow answers, and older lockfiles
  reference `dinero.js@alpha`, `2.0.0-alpha.x`, and pre-stable import paths. When wiring v2
  today, pin `>=2.0.0` explicitly and ignore alpha-era guidance about package layout — it moved.
- **v1 → v2 is a rewrite, not an upgrade.** There is no codemod. Chainable `.add().multiply()`
  calls become nested/piped free functions, formatting moves out of the library, and currency
  objects replace inline `precision` numbers. Budget for a real migration, not a version bump.
- **Formatting is your job.** Dinero intentionally omits locale formatting[^6]; you pair
  `toDecimal` with `Intl.NumberFormat`. This is a feature (no bundled ICU weight) but surprises
  teams expecting `.toFormat('$0,0.00')` like v1 offered.
- **Scale mismatches fail loudly-ish.** Mixing values of different scale or currency throws;
  the footgun is silently-diverging scale after `allocate`/division, which produces correct but
  higher-precision results you must `transformScale` back down before display.
- **Rounding is explicit.** `allocate` distributes remainders so splits sum back to the original
  (no lost pennies), but general division takes a rounding mode you must choose deliberately.
- **Bundle vs. correctness.** The `number` calculator is lighter; reach for `bigint` only when
  amounts or scales genuinely overflow safe integers, since it enlarges the bundle and changes
  input types.

## When to Use / When Not

**Use when:**
- You do any arithmetic on money and want integers, immutability, and no float drift.
- You are on TypeScript and want currency/scale encoded in the type system.
- You need multi-currency, non-decimal, or high-precision (sub-cent) values.
- Bundle size matters and you want to import only the operations you use.

**Avoid when:**
- You only ever display a preformatted price string — `Intl.NumberFormat` alone is enough.
- You want a batteries-included `.format()` API; that is v1's shape, and v1 is effectively frozen.
- You need decimal arithmetic beyond money (scientific, arbitrary-precision math) — use a decimal
  library instead of bending currency semantics around it.
- Long-term-stable dependency risk is a hard constraint and the 2023–2026 dormancy worries you.

## Alternatives

- dinerojs/dinero.js v1 — the OO/chainable predecessor; still installable, still works, but no
  new development. Use when a legacy codebase already depends on it and migration isn't worth it.
- MikeMcl/big.js — tiny arbitrary-precision decimals. Use when you want raw decimal math and will
  handle currency, rounding, and formatting yourself.
- MikeMcl/decimal.js — heavier, more complete decimal type. Use when you need trig/exponents on
  decimals, not just money.
- js-money/js-money — Fowler-style Money pattern, simpler and mutable-ish. Use when you want a
  minimal Money object and don't need tree-shaking or pluggable precision.
- Intl.NumberFormat (built-in) — no dependency. Use when you only format, never compute.

## History

| Version | Date | Notes |
|---------|------|-------|
| v1.0.0 | 2018-04-17 | First stable release. OO chainable API, built-in formatting[^2]. |
| v1.9.1 | ~2020 | Last significant v1 line; effectively the frozen "old" API. |
| v2.0.0-alpha.1 | 2021-07-12 | Functional rewrite begins: pure functions, TS-first, tree-shakeable[^3]. |
| v2.0.0-alpha.14 | 2023-02-27 | Last alpha before a ~3-year dormant gap. |
| v2.0.0-alpha.15 | 2026-01-31 | Development resumes after the hiatus. |
| v2.0.0 | 2026-03-02 | v2 finally stable, ~4.5 years after the first alpha[^3]. |
| v2.0.2 | 2026-03-13 | Current stable at time of writing. |

## References

[^1]: Dinero.js core concepts — amount, currency, scale, immutability. https://dinerojs.com/core-concepts/amount
[^2]: dinerojs/dinero.js repository, created 2018-03-10; v1.0.0 released 2018-04-17. https://github.com/dinerojs/dinero.js/releases
[^3]: Release timeline: v2.0.0-alpha.1 (2021-07-12) → alpha.14 (2023-02-27) → alpha.15 (2026-01-31) → v2.0.0 stable (2026-03-02). https://github.com/dinerojs/dinero.js/releases
[^4]: Precision and large numbers — number vs. bigint calculator. https://dinerojs.com/guides/precision-and-large-numbers
[^5]: Currency and non-decimal / multi-subdivision support. https://dinerojs.com/core-concepts/currency
[^6]: FAQ — "Why is there no currency formatting?" https://dinerojs.com/faq/why-no-currency-formatting

## Tags

typescript, javascript, money, currency, decimal, immutable, tree-shakeable, finance, arithmetic, functional, npm-package
