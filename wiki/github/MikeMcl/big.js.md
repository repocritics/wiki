# MikeMcl/big.js

> A ~6 KB arbitrary-precision decimal library for JavaScript — the deliberately minimal one in the bignumber.js/decimal.js family.

[GitHub repo](https://github.com/MikeMcl/big.js) ·
[Official website](http://mikemcl.github.io/big.js) ·
[License: MIT](https://github.com/MikeMcl/big.js/blob/main/LICENCE.md)

## Overview

big.js is a single-file library for exact decimal arithmetic in JavaScript. It exists because IEEE-754 doubles cannot represent most decimal fractions: `0.3 - 0.1` evaluates to `0.19999999999999998`, which is unacceptable for money, invoicing, and anywhere a displayed number must match a hand calculation. big.js stores numbers as a decimal coefficient, exponent, and sign, and performs arithmetic digit-by-digit so results are exact (with the exception of division, square root, and negative-exponent powers, which are rounded to a configured number of decimal places).

Written by Michael Mclaughlin and first published in 2012, big.js is the smallest member of a three-library family by the same author: it is the "little sister" to `bignumber.js` and `decimal.js`.[^1] The defining tradeoff is scope. big.js does one thing — base-10 decimal arithmetic with a handful of methods — and refuses almost everything else: no non-integer exponents, no logarithms or trigonometry, no non-decimal bases, no `NaN`/`Infinity` sentinels (invalid input throws). That minimalism is the product. If you need transcendental functions or significant-digit precision you are expected to move up to `decimal.js`; if you need other number bases, to `bignumber.js`.

The library targets ECMAScript 3 with zero dependencies, so it runs unchanged in ancient browsers, modern runtimes, Node.js, and Deno. At roughly 6 KB minified it is small enough to include without a second thought, which is a large part of why it remains widely depended upon more than a decade after release.[^2]

## Getting Started

```bash
npm install big.js
# optional TypeScript types (community-maintained, not bundled)
npm install --save-dev @types/big.js
```

```javascript
import Big from 'big.js';   // or: const Big = require('big.js')

// The float trap big.js exists to solve:
0.3 - 0.1;                  // 0.19999999999999998
new Big('0.3').minus('0.1').toString();   // "0.2"

// Immutable, chainable, compare with methods (not ===):
const price = new Big('19.99');
const total = price.times(3).plus('0.50');   // "60.47"
total.gt('60');             // true

// Division is rounded to Big.DP decimal places using Big.RM:
Big.DP = 10;                // default is 20
Big.RM = Big.roundHalfUp;   // default rounding mode
new Big(2).div(3).toString();   // "0.6666666667"
```

Prefer string inputs. Passing a primitive number (`new Big(0.1 + 0.2)`) feeds an already-lossy double into the constructor — big.js faithfully preserves whatever imprecision the float already carried. Set `Big.strict = true` to make number-primitive construction (and lossy `toNumber()`) throw, forcing explicit strings.

## Architecture / How It Works

A Big number is a plain object with three enumerable properties: `c` (coefficient, an array of single decimal digits / the significand), `e` (exponent, the power-of-ten position of the first digit), and `s` (sign, `1` or `-1`). `new Big(-123.456)` yields `c = [1,2,3,4,5,6]`, `e = 2`, `s = -1`. This decimal-native representation is why big.js is exact where floating point is not: there is never a binary-to-decimal conversion step to lose precision.

Arithmetic is implemented as schoolbook algorithms over the coefficient arrays. `plus`, `minus`, and `times` are always exact — their results carry however many digits the operation produces. Only operations that can produce a non-terminating decimal are rounded: `div`, `sqrt`, and `pow` with a negative exponent. Those consult two constructor-level properties — `DP` (decimal places, default 20) and `RM` (rounding mode) — and round accordingly. Rounding modes are exposed as named constants (`roundDown`, `roundHalfUp`, `roundHalfEven`, `roundUp`, i.e. 0–3).

Two design decisions shape everyday use. First, precision is measured in **decimal places**, not significant digits — this is the primary behavioral difference from `decimal.js`, which counts significant digits. Second, `DP` and `RM` live on the constructor, so they are effectively shared mutable configuration for every Big number created from it. For code that needs independent settings, big.js lets you build additional constructors, each with its own `DP`/`RM`, via its factory export, so a money module and a scientific module can coexist without stepping on each other's rounding.

Every method returns a new Big number; instances are never mutated. Comparison must go through methods (`eq`, `gt`, `gte`, `lt`, `lte`, `cmp`) because `===` compares object identity and `<`/`>` would coerce to primitives. The library ships as `big.js` (UMD/CommonJS + global) and `big.mjs` (ES module); `@types/big.js` in DefinitelyTyped provides TypeScript definitions and is not part of the package itself.

## Production Notes

- **`Big.DP` / `Big.RM` are global mutable state.** A library, a dependency, or a stray line that sets `Big.DP = 2` silently changes rounding for every division in the whole process. If you share a Big constructor across modules, treat these as global config: set them once at startup, or create isolated constructors for subsystems with different needs.
- **Decimal places, not significant digits.** `Big.DP = 20` means 20 digits *after the point*. For very large or very small magnitudes this is not the same as 20 significant figures. Teams migrating from `decimal.js` routinely get this wrong.
- **Division rounds; addition/multiplication do not.** `a.times(b)` can grow the coefficient without bound (exact), while `a.div(b)` is capped at `DP`. Long chains of exact multiplications on high-precision inputs can produce surprisingly long coefficients and the associated memory/CPU cost.
- **No `NaN` or `Infinity`.** Invalid input, division by zero, `sqrt` of a negative, and `0` to a negative power throw a `[big.js]` error instead of returning a sentinel. Production code doing arithmetic on user input needs try/catch or pre-validation.
- **`pow` takes integer exponents only.** There is no `x^0.5` via `pow` (use `sqrt`), no `exp`, `ln`, `log`, or trigonometry anywhere. Reaching for those means you have outgrown big.js and want `decimal.js`.
- **`toNumber()` can silently lose precision** unless `Big.strict` is on, in which case it throws on an imprecise conversion. Don't round-trip through `Number` in a calculation pipeline.
- **Value stability.** big.js is mature and slow-moving; the v7 release (2025) was the first tag in nearly five years, so pinning a major version and forgetting about it is a reasonable strategy. Breaking changes across majors have historically been small and well-documented in the CHANGELOG.

## When to Use / When Not

**Use when:**
- You need exact base-10 decimal math for money, tax, billing, or reporting where displayed totals must be correct to the cent.
- You want the smallest possible dependency (~6 KB) with no transitive packages and broad runtime support.
- Your operations are the basics: add, subtract, multiply, divide, `sqrt`, integer `pow`, compare, round, and formatted output (`toFixed`/`toExponential`/`toPrecision`).

**Avoid when:**
- You need transcendental functions (`exp`, `ln`, `sin`, non-integer powers) — use `decimal.js`.
- You work in non-decimal bases or need base-2..64 I/O and bitwise-style operations — use `bignumber.js`.
- You only deal with integers — native `BigInt` is built in, faster, and needs no library.
- You are building a money-specific domain model and want currency/allocation semantics rather than raw numbers — a dedicated money library sits better on top.

## Alternatives

- MikeMcl/decimal.js — same author; significant-digit precision plus `exp`/`ln`/`sin`/`pow` with fractional exponents. Use when you need transcendental functions or non-terminating precision.
- MikeMcl/bignumber.js — same author; supports bases 2–64 and a larger API. Use when you need non-decimal bases or the extra methods and can spend the extra bytes.
- native BigInt — built into JavaScript. Use when your values are integers only; no decimals, no library.
- dinerojs/dinero.js — money-specific, immutable, minor-unit integer model. Use when you want currency, formatting, and allocation semantics rather than a general number type.
- royNiladri/js-big-decimal — an alternative arbitrary-precision decimal implementation. Use when evaluating options and you want a second reference point on API and size.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2012-11-07 | First published to GitHub.[^1] |
| 3.0.0 | 2014-12-10 | Early API consolidation. |
| 5.0.0 | 2017-10-13 | Major release; API and rounding-mode cleanups. |
| 6.0.0 | 2020-09-25 | Major release; `Big.strict` mode, `toNumber`, config refinements. |
| 7.0.0 | 2025-04-21 | Major release after a ~5-year gap; ES module (`big.mjs`) shipped. |
| 7.0.1 | 2025-04-22 | Latest tag; patch to the v7 line.[^2] |

## References

[^1]: big.js README and repository — origin, family relationship to bignumber.js/decimal.js, ES3/no-dependency design. https://github.com/MikeMcl/big.js
[^2]: npm package `big.js` (v7.0.1) and API documentation — size, methods, `DP`/`RM`/strict semantics. https://www.npmjs.com/package/big.js and http://mikemcl.github.io/big.js/

## Tags

javascript, decimal, arbitrary-precision, bignumber, math, floating-point, money, es3, zero-dependency, browser, nodejs
