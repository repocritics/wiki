# josdejong/mathjs

> An extensive math library for JavaScript and Node.js: mixed-type arithmetic, an expression parser, and light symbolic computation, all in pure JS.

[GitHub repo](https://github.com/josdejong/mathjs) ·
[Official website](https://mathjs.org) ·
[License: Apache-2.0](https://github.com/josdejong/mathjs/blob/develop/LICENSE)

## Overview

mathjs is a single-author library started by Jos de Jong in 2013[^1] that tries to be the "batteries-included" math layer JavaScript never shipped. Where the built-in `Math` object gives you `float64` and a handful of functions, mathjs adds arbitrary-precision `BigNumber`, exact `Fraction`, `Complex`, physical `Unit`, `bigint`, and dense/sparse `Matrix` types, and — crucially — lets you multiply, add, and compose across all of them. On top of the numeric core sits a full expression parser (`evaluate('12.7 cm to inch')`) and a modest computer-algebra layer (`derivative`, `simplify`, `rationalize`).

The defining design bet is that every function is a *typed function*: one name, many signatures, with automatic type coercion and dynamic dispatch. This is what makes `multiply(aBigNumber, aComplex)` "just work" and what makes the library extensible with your own data type in a few lines. It is also the source of the two things people complain about most: runtime dispatch overhead and bundle size. mathjs is broad rather than fast, and it is honest about that — it targets correctness and ergonomics over raw throughput, and it is not a replacement for a native/BLAS numerical stack.

As of 2026 the project is maintained but no longer moving quickly: ~15k stars, one dominant author, and a release cadence measured in months, not weeks[^2]. The last push was in mid-2026, so it is alive, but treat it as a stable, feature-complete utility rather than a fast-evolving framework.

## Getting Started

```bash
npm install mathjs
```

```js
import { evaluate, derivative, bignumber, unit, chain } from 'mathjs'

evaluate('sqrt(-4)')              // Complex 2i
evaluate('det([-1, 2; 3, 1])')   // -7
evaluate('12.7 cm to inch')      // 5 inch  (Unit)

derivative('x^2 + x', 'x').toString()   // '2 * x + 1'

bignumber(0.1).plus(0.2).toString()     // '0.3'  (not 0.30000000000000004)

chain(3).add(4).multiply(2).done()      // 14
```

For untrusted input, do **not** use the top-level `evaluate` — build a restricted instance instead (see Production Notes).

## Architecture / How It Works

Two ideas carry the whole library:

1. **`typed-function`** — a companion library (also by de Jong)[^3] that builds one function out of many per-type implementations. `add` is not a function so much as a dispatch table: `add(number, number)`, `add(BigNumber, BigNumber)`, `add(Complex, Complex)`, plus conversion rules that let mismatched arguments be coerced (e.g. `number → Complex`). Adding support for a new type means registering another signature and a conversion; every function that internally calls `add` inherits the new behavior.

2. **Factory functions + dependency injection** — since the v6 rewrite (2019)[^4], each function is defined as a factory that declares its dependencies. `math.create(...)` assembles an instance by wiring factories together. This is what enables tree-shaking: `math.create({ addDependencies })` yields an `add` that pulls in only what it needs, instead of the whole library. There are two build flavors — `any` (full multi-type functions) and `number` (lightweight, `number`-only).

Underneath, the numeric types are mostly delegated to focused third-party libraries: `BigNumber` is decimal-based via decimal.js, `Fraction` via fraction.js, `Complex` via complex.js. Sparse-matrix operations are a JavaScript port of Tim Davis's CSparse, which is vendored under LGPL-2.1+ (a different license from the Apache-2.0 project itself[^5] — relevant if you audit dependency licenses). The expression parser compiles a string into an AST of nodes (`OperatorNode`, `FunctionNode`, `SymbolNode`, …) which can be transformed, `.toString()`'d back, or compiled to an evaluatable form — this AST is what the symbolic functions (`simplify`, `derivative`, `rationalize`) walk.

The cost of this generality is indirection: a single `add` call goes through typed dispatch, possible type conversion, and the injected implementation before doing arithmetic. For scalar-heavy hot loops this is measurably slower than plain `+`.

## Production Notes

**Bundle size is the number-one footgun.** Importing the whole library (`import * as math` or `import { evaluate } from 'mathjs'`) pulls in a large graph — hundreds of KB minified — because the expression parser and symbolic layer drag in nearly everything. Tree-shaking works only if you use the factory/`create` path with explicit `*Dependencies` objects, which is verbose and easy to get wrong. If you need `add`/`multiply` on plain numbers in a browser bundle, mathjs is often the wrong tool; if you need the parser, budget for the weight.

**`evaluate()` on untrusted input is a security surface.** The parser has historically been the subject of sandbox-escape advisories where crafted expressions reached JavaScript internals (constructor chains, `import`, function creation)[^6]. mathjs has hardened this repeatedly, but the safe posture is: never pass user-controlled strings to the global `math.evaluate`. Create a scoped instance, disable dangerous functions (`import`, `createUnit`, `evaluate`, `parse`, `simplify`, `derivative`) via `math.import({...}, { override })` or the documented "safe evaluation" pattern, and pass an explicit scope object. Assume any published mathjs version *might* have an undiscovered escape and keep it patched.

**BigNumber is decimal, not binary, and it is slow.** decimal.js gives you exact decimal arithmetic with configurable precision (`math.config({ number: 'BigNumber', precision: 64 })`), which is the right call for money and for avoiding `0.1 + 0.2` drift — but it is orders of magnitude slower than native `number`. Don't switch the global config to BigNumber and then run large numeric loops.

**Matrices are pure JS.** Dense and sparse matrices work and are correct, but this is not NumPy/BLAS. Large linear-algebra workloads (big `inv`, `eig`, `lusolve`) will be slow and memory-hungry compared to native/WASM numeric stacks. mathjs is a convenience layer, not a compute kernel.

**Upgrade friction is mostly at the majors.** The v6 (2019) refactor changed the recommended import style (factory/`create`, ESM-first). Major bumps have periodically dropped old Node versions and tightened types. TypeScript definitions are hand-maintained in `types/index.d.ts` and can lag new functions — verify a function's typings exist before relying on them in strict TS.

## When to Use / When Not

**Use when:**
- You need an expression parser to evaluate math strings (calculators, formula fields, notebooks).
- You want one API that spans numbers, big/decimal numbers, fractions, complex numbers, and units with automatic coercion.
- You need light symbolic work — derivatives, simplification — without a full CAS.
- You value breadth and a stable, well-documented surface over raw speed.

**Avoid when:**
- Bundle size is critical and you only need a couple of operations — import the underlying single-purpose libs (decimal.js, fraction.js) directly.
- You're doing heavy numerical/linear-algebra compute — reach for a native/WASM stack.
- You need a hardened, audited sandbox for arbitrary untrusted expressions — the parser is not designed as a security boundary.
- You need serious symbolic algebra (integration, equation solving) — this is a light CAS, not Sympy.

## Alternatives

- MikeMcl/decimal.js — arbitrary-precision decimals only; use when you just need exact decimal arithmetic without the parser or the weight.
- rawify/Fraction.js — exact rational numbers only; use when fractions are all you need.
- silentmatt/expr-eval — a small, parser-only expression evaluator; use when you want to eval math strings and nothing else.
- stdlib-js/stdlib — broad numerical-computing standard library for JS; use when you need statistics/ndarray primitives more than a parser.
- nerdamer or Algebrite — heavier symbolic/CAS engines; use when derivatives-and-simplify isn't enough and you need real algebra.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2013-02 | Project started by Jos de Jong[^1]. |
| 1.0 | 2015 | First stable release; expression parser, matrices, units[^2]. |
| 3.0 | 2016 | typed-function-based core; broad type support. |
| 6.0 | 2019 | Factory-function / dependency-injection rewrite; ESM-first, tree-shakeable imports[^4]. |
| 10.0 | 2022 | Continued major line; Node/type maintenance. |
| 12.0 | 2023 | — |
| 14.0 | 2024–25 | Current major line as of 2026[^2]. |

Exact release dates and per-version details are on the project's history page[^2]; the major-version notes above are approximate.

## References

[^1]: Repository created 2013-02-15; author Jos de Jong. https://github.com/josdejong/mathjs
[^2]: mathjs release history / changelog. https://mathjs.org/history.html
[^3]: `typed-function` — the dynamic type-dispatch library underpinning mathjs. https://github.com/josdejong/typed-function
[^4]: mathjs docs, "Custom bundling" / factory functions and `math.create`. https://mathjs.org/docs/custom_bundling.html
[^5]: mathjs README, License section — Apache-2.0 project vendoring an LGPL-2.1+ CSparse port. https://github.com/josdejong/mathjs#license
[^6]: mathjs docs, "Expression parser — Security". https://mathjs.org/docs/expressions/security.html

## Tags

javascript, math, expression-parser, symbolic-computation, bignumber, complex-numbers, matrices, units, computer-algebra, numerical-computing
