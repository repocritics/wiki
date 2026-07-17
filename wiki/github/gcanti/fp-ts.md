# gcanti/fp-ts

> Typed functional programming for TypeScript — Option/Either/Task and the type-class hierarchy on top of emulated higher-kinded types.

[GitHub repo](https://github.com/gcanti/fp-ts) ·
[Official website](https://gcanti.github.io/fp-ts/) ·
[License: MIT](https://github.com/gcanti/fp-ts/blob/master/LICENSE)

## Overview

fp-ts is a library that ports the vocabulary of Haskell, PureScript, and Scala's
functional core to TypeScript: data types (`Option`, `Either`, `IO`, `Task`,
`Reader`, and their stacked variants like `ReaderTaskEither`) and the type-class
hierarchy above them (`Functor`, `Applicative`, `Monad`, `Traversable`,
`Semigroup`, `Monoid`). Its distinctive technical contribution is emulating
higher-kinded types — which TypeScript does not support natively — via a
defunctionalization trick, so that abstractions like `Monad` can be written once
and instantiated for any type constructor[^1].

The project was authored by Giulio Canti and first published in 2017. It went
through one large breaking pivot (v1 → v2) that changed the entire API shape, and
is now in a settled maintenance state. As of 2026 the repository carries roughly
11.5k stars and 512 forks; the last substantive push was April 2026, and the
issue tracker sits near 190 open items — numbers consistent with a mature library
in wind-down rather than active feature growth.

The defining fact for anyone evaluating fp-ts today is the handoff: the
maintainers have publicly announced that fp-ts is joining the Effect-TS ecosystem,
and that Effect should be regarded as the successor — effectively "fp-ts v3"[^2].
New projects are steered toward `effect`; fp-ts remains supported and widely
depended-upon, but is not where new capability lands.

## Getting Started

```bash
npm install fp-ts
```

fp-ts is designed to be consumed with TypeScript's `strict` flag on; v2.x requires
TypeScript 3.5 or later[^3]. Keep exactly one copy in your dependency tree —
multiple installed versions are known to make `tsc` hang during compilation. Check
with `npm ls fp-ts` and confirm all but one are `deduped`.

```ts
import { pipe } from "fp-ts/function";
import * as O from "fp-ts/Option";

const parse = (s: string): O.Option<number> => {
  const n = Number(s);
  return isNaN(n) ? O.none : O.some(n);
};

const result = pipe(
  parse("42"),
  O.map((n) => n * 2),
  O.getOrElse(() => 0)
); // 84
```

The `pipe`/`flow` combinators plus data-last, curried functions are the idiomatic
v2 style. You import a module namespace (`* as O`) and thread values through it,
rather than calling methods on a value.

## Architecture / How It Works

The core mechanism is **lightweight higher-kinded types**. TypeScript cannot write
`F<A>` where `F` is a generic parameter, so fp-ts uses the technique from Yallop &
White's "Lightweight Higher-Kinded Polymorphism"[^1]: each type constructor
registers a unique string URI in a global `URItoKind` interface via declaration
merging, and generic code refers to constructors indirectly through that registry
(`Kind<F, A>`). This is why every fp-ts data type module declares an `URI` constant
and augments a shared interface — it is the machinery that makes `Monad<F>` express
as a real, reusable abstraction.

On top of that sit two layers:

- **Type classes** — `Functor`, `Apply`, `Applicative`, `Chain`, `Monad`,
  `Foldable`, `Traversable`, `Semigroup`, `Monoid`, `Eq`, `Ord`, `Show`, and more.
  Each is an interface parameterized over a registered URI.
- **Instances / data types** — concrete modules (`Option`, `Either`, `Task`,
  `IO`, `Reader`, `State`, and the monad-transformer-style stacks
  `TaskEither`, `ReaderTaskEither`, `StateReaderTaskEither`) that satisfy those
  type classes and expose data-last functions.

The single largest architectural event in fp-ts history is the **v1 → v2 rewrite**.
v1 was method/fluent-based (`option.map(f).getOrElse(x)`) with a class-heavy design.
v2 abandoned that for standalone, curried, data-last functions composed through
`pipe`, which tree-shake far better and interact more predictably with type
inference. The two styles are not source-compatible; migrating a codebase is a
mechanical but pervasive rewrite. A `fp-ts-codegen`/codemod path existed but most
teams migrated by hand.

## Production Notes

**Single-version discipline is a real footgun.** The README's warning is not
boilerplate: two versions of fp-ts in `node_modules` produce distinct, unmergeable
`URItoKind` declarations, which manifests as `tsc` hanging or as bewildering type
errors where an `Option` from one copy is not assignable to an `Option` from the
other. This bites hardest when a transitive dependency pins a different major.

**Type-error legibility.** Because so much rides on inference through curried,
higher-kinded generics, a small mistake (wrong argument order, a missing `pipe`
step) can produce long, deeply-nested error messages that name internal type
aliases. This is the most common complaint from teams onboarding non-FP engineers,
and it is inherent to the HKT emulation, not a bug.

**Bundle size / tree-shaking.** v2's per-function, namespace-import design tree-shakes
well — you pay for the modules you import. Importing whole namespaces (`import * as O`)
is fine with a modern bundler; the concern is more about which combinators you pull
in than the library's total footprint.

**No runtime effect system.** fp-ts `Task`/`TaskEither` are thin lazy `Promise`
wrappers. There is no fiber runtime, no structured concurrency, no interruption, no
dependency-injection layer, and no built-in retry/scheduling. Those are exactly the
gaps Effect fills — which is the practical reason the ecosystem is consolidating
there.

**Migration pressure.** Given the Effect handoff[^2], the operative question for a
new codebase is not "which fp-ts version" but "fp-ts or Effect." Choosing fp-ts in
2026 is choosing a stable, smaller, well-understood library whose upstream momentum
has moved elsewhere. Existing fp-ts code keeps working; there is no forced upgrade,
but expect new learning material and community energy to point at Effect.

## When to Use / When Not

**Use when:**
- You want just `Option`/`Either`/`Task` and the type-class discipline, without
  adopting a full runtime.
- You already have an fp-ts codebase and it works — there is no urgency to rewrite.
- You value a small, stable dependency and your team is comfortable with data-last
  FP style.
- You're teaching or learning the type-class hierarchy in a TypeScript setting.

**Avoid when:**
- You're starting fresh and want the maintained path — use Effect.
- You need concurrency, fibers, interruption, resource safety, or dependency
  injection at runtime.
- Your team has no FP background and error-message legibility matters more than
  purity.
- You need a library with active feature development and long-term roadmap growth.

## Alternatives

- Effect-TS/effect — the designated successor; use when you want a maintained
  library with a real effect runtime (fibers, DI, structured concurrency).
- gigobyte/purify (purify-ts) — use when you want `Maybe`/`Either` with a lighter,
  more approachable API and no higher-kinded-type machinery.
- true-myth/true-myth — use when you only need `Maybe` and `Result` with a gentle,
  well-documented surface for an FP-curious team.
- ramda/ramda — use when you want data-last functional utilities and composition
  but not type classes or algebraic data types.
- gvergnaud/ts-pattern — use when your real goal is exhaustive pattern matching
  rather than the full FP abstraction stack.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-01 | First published; class/method-based (v1 line). |
| 2.0.0 | 2019-06 | Full rewrite to data-last, pipeable functions; requires TS 3.5+[^3]. |
| 2.x | 2019–2026 | Long stable maintenance line; the current release series. |
| — | announced | fp-ts joins Effect-TS; Effect positioned as successor ("v3")[^2]. |

## References

[^1]: Jeremy Yallop and Leo White, "Lightweight Higher-Kinded Polymorphism"
(FLOPS 2014) — the defunctionalization technique fp-ts uses to emulate higher-kinded
types. https://www.cl.cam.ac.uk/~jdy22/papers/lightweight-higher-kinded-polymorphism.pdf
[^2]: "A bright future for Effect" — announcement that fp-ts is merging into the
Effect-TS ecosystem, with Effect as the successor to fp-ts v2.
https://dev.to/effect/a-bright-future-for-effect-455m
[^3]: fp-ts README, "TypeScript compatibility" — v2.0.x+ requires TypeScript 3.5+.
https://github.com/gcanti/fp-ts#typescript-compatibility

## Tags

typescript, functional-programming, algebraic-data-types, higher-kinded-types, monads, option-either, type-classes, effect-ts, library, maintenance-mode
