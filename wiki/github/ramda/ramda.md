# ramda/ramda

> A functional JavaScript utility library built around auto-currying and data-last argument order, designed for point-free composition without mutation.

[GitHub repo](https://github.com/ramda/ramda) ·
[Official website](https://ramdajs.com) ·
[License: MIT](https://github.com/ramda/ramda/blob/master/LICENSE.txt)

## Overview

Ramda is a utility library in the lineage of Underscore and Lodash, but with two deliberate inversions that define everything about it. First, every function is **automatically curried**, so any function can be partially applied by omitting trailing arguments. Second, arguments are arranged **data-last** — the collection or value being operated on is the final parameter, not the first. Together these make point-free pipelines (`R.pipe(R.map(f), R.filter(g))`) the natural way to write code, rather than an awkward opt-in. Ramda never mutates its inputs; every operation returns a new value.

The library was started in 2013 by Scott Sauyet and Michael Hurley (Buzz de Cafe) and has been maintained by a small group ever since[^1]. Its defining tension is that it optimizes for API purity over almost everything else — the README itself says "the API is king" and that the authors "sacrifice a great deal of implementation elegance for even a slightly cleaner API." That choice produces elegant composition but pushes real costs onto consumers: currying adds runtime overhead, point-free style produces opaque stack traces, and the data-last + variadic + placeholder design is genuinely hard to type in TypeScript.

Notably, after more than a decade Ramda has never shipped a 1.0. It remains a `0.x` project (0.31.3 is the current line as of mid-2026), and while the core API is stable in practice, that version number is an honest signal: the maintainers reserve the right to break things, and release cadence has slowed considerably compared to its 2015–2018 peak.

## Getting Started

```bash
npm install ramda
```

```javascript
import * as R from "ramda";

// data-last + auto-curry → point-free pipeline
const activeNames = R.pipe(
  R.filter(R.propEq(true, "active")),
  R.map(R.prop("name")),
  R.sortBy(R.toLower)
);

activeNames([
  { name: "Bea", active: true },
  { name: "Al",  active: false },
  { name: "Cy",  active: true },
]); // => ["Bea", "Cy"]
```

For versions above 0.25 there is no default export: use `import * as R from "ramda"` or cherry-pick named functions (`import { map, filter } from "ramda"`)[^2]. Ramda also publishes to deno.land and is available via cdnjs / jsDelivr for direct browser use, where it attaches a global `R`.

## Architecture / How It Works

The mechanical core is `_curryN` plus a per-arity family (`_curry1`, `_curry2`, `_curry3`). Every public function is wrapped so that calling it with fewer than its full argument count returns a new curried function instead of executing. This is what allows `R.map(fn)` to be a reusable transformer and `R.add(1)` to be an increment function. The **placeholder** `R.__` lets you skip a leading argument and supply it later — e.g. `R.subtract(R.__, 1)` — which is the escape hatch for the cases where data-last ordering is inconvenient.

Because data is always last and functions are always curried, `R.pipe` and `R.compose` are the spine of idiomatic Ramda. `pipe` reads left-to-right; `compose` reads right-to-left. Both build a single function from a sequence of unary transformations.

Two more subsystems are worth knowing:

- **Lenses** (`R.lens`, `R.view`, `R.set`, `R.over`, plus `R.lensProp` / `R.lensPath` / `R.lensIndex`) provide composable, immutable getter/setter pairs for reaching into nested structures without mutation. They are a functional-optics implementation over plain JS objects.
- **Transducers.** Many list functions (`map`, `filter`, `take`) double as transducers, and `R.transduce` / `R.into` let you compose transformations that run in a single pass without allocating intermediate arrays. This is the one place Ramda addresses the allocation cost of chained operations.

Crucially, Ramda operates on **plain JavaScript objects and arrays** — it does not introduce persistent/immutable data structures the way Immutable.js does. "Immutable" here means "does not mutate inputs," achieved by copying. There is no structural sharing, so operations on large objects copy proportionally to their size.

## Production Notes

**TypeScript support is the single biggest friction point.** Types live externally in `@types/ramda` (DefinitelyTyped), not in the package. Ramda's combination of heavy currying, placeholders, and variadic overloads is close to the worst case for TypeScript inference: `R.pipe` / `R.compose` are typed via a finite stack of fixed-arity overloads, inference through polymorphic and curried functions frequently collapses to `any` or requires manual type annotations, and updates to the types lag the runtime library. Teams that are TypeScript-first often reach for Remeda or rambda specifically to escape this.

**Debugging point-free code is hard.** A deeply composed `R.pipe(...)` throws with a stack trace full of internal `_curryN` frames and no reference to your own function names. There are no intermediate variables to inspect. The common mitigation is `R.tap(x => console.log(x))` inserted into a pipeline, but this remains a real cost of the style.

**Bundle size and tree-shaking.** Importing from `"ramda"` can pull in far more than you use because functions share internal helpers. Tree-shaking effectiveness varies by bundler: Rollup handles it well, Webpack historically needed `babel-plugin-ramda` (to rewrite `import { x } from "ramda"` into deep `ramda/src/x` imports) or careful scope-hoisting configuration to actually drop unused code[^3]. Do not assume named imports alone give you a minimal bundle.

**Currying has a runtime cost.** Every call goes through wrapper logic that inspects argument counts. For most application code this is irrelevant, but in hot loops over large collections, native `Array.prototype` methods or hand-written loops are measurably faster. Ramda lists performance as a goal, but the currying layer is a fixed tax.

**Maintenance velocity has dropped.** The project is stable and still receives fixes, but it is no longer under active feature development the way it was mid-decade. Evaluate it as a mature, slow-moving dependency rather than a growing one.

## When to Use / When Not

**Use when:**
- You are writing functional, pipeline-heavy JavaScript and want currying and data-last ordering as first-class defaults.
- You value never mutating inputs and want composable lenses and transducers out of the box.
- Your codebase is JavaScript (not strict TypeScript), where the type-inference limitations don't bite.

**Avoid when:**
- You are TypeScript-first and want precise inference through composition — Remeda or rambda will hurt less.
- You need persistent immutable collections with structural sharing for large data — use Immutable.js or Immer.
- You mostly want a grab-bag of utilities with familiar data-first ergonomics — Lodash is more idiomatic and more widely known.
- Bundle size is critical and you can't invest in build configuration to tree-shake it properly.

## Alternatives

- lodash/lodash — the mainstream utility library; data-first, larger surface, `lodash/fp` offers a curried data-last variant closer to Ramda's model. Use when you want familiarity and ecosystem ubiquity.
- remeda/remeda — TypeScript-first functional utilities with strong inference and both data-first and data-last forms. Use when types matter most.
- selfrefactor/rambda — a smaller, faster, mostly API-compatible reimplementation with bundled TypeScript types. Use when you want Ramda's style with less weight and better TS.
- immutable-js/immutable — persistent immutable data structures with structural sharing. Use when the concern is efficient immutable collections, not function composition.
- Effect-TS/effect — a full typed functional ecosystem (formerly fp-ts territory). Use when you want principled, typed FP abstractions well beyond utilities.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2013-06 | Project started by Scott Sauyet and Michael Hurley[^1]. |
| 0.25.0 | 2018 | Removed the default export; consumers must use `import * as R` or named imports[^2]. |
| 0.27.x | 2020–2021 | Deno / nest.land distribution referenced in official docs. |
| 0.31.3 | 2026 | Current release line (per official CDN references). |

Ramda has intentionally never released a 1.0 in over a decade of development.

## References

[^1]: Ramda README and project history. https://github.com/ramda/ramda
[^2]: Ramda README, "Note for versions > 0.25" — default export removed. https://github.com/ramda/ramda#usage
[^3]: Tree-shaking discussion and examples for Ramda across bundlers. https://github.com/scabbiaza/ramda-webpack-tree-shaking-examples

## Tags

javascript, functional-programming, utility-library, currying, point-free, immutability, lenses, transducers, composition, data-last
