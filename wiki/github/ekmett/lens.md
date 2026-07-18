# ekmett/lens

> The kitchen-sink optics library for Haskell — lenses, prisms, folds, and
> traversals as ordinary composable functions, at the price of a heavy
> dependency tree and a famously large API surface.

[GitHub repo](https://github.com/ekmett/lens) ·
[Official website](http://lens.github.io/) ·
[License: BSD-2-Clause](https://github.com/ekmett/lens/blob/master/LICENSE)

## Overview

`lens` is Edward Kmett's 2012 library of first-class functional references[^1].
It generalizes "get/set a record field" into a hierarchy of optics: `Lens`
(exactly one target), `Prism` (zero-or-one, invertible), `Traversal` (zero or
more), `Fold` (read-only many), `Getter`, `Setter`, and `Iso`. All of them are
encoded as plain polymorphic functions (the van Laarhoven encoding[^2]), which
is the library's central trick: optics compose with `Prelude`'s `(.)`, in the
"path-like" order imperative programmers expect, and the subtyping between
optic kinds falls out of typeclass constraints rather than any conversion
functions.

Its GitHub numbers (2,094 stars, 273 forks) understate its position: Haskell
activity centers on Hackage, where `lens` is among the most heavily
reverse-depended-upon packages in the ecosystem and effectively defined the
vocabulary — `^.`, `.~`, `%~`, `makeLenses` — that every other Haskell optics
library either adopts or deliberately reacts against. The repo is actively
maintained (pushed July 2026), with maintenance long since broadened beyond
Kmett to co-maintainers who track new GHC releases.

The defining tension: `lens` chooses maximal power and zero abstraction
overhead over approachability. The same encoding that makes optics compose
with `(.)` also produces rank-2 type synonyms, hostile type errors, and an API
of several hundred functions and operators. Whole competing libraries
(`optics`, `microlens`) exist purely to walk back one or the other of these
costs[^3].

## Getting Started

Add `lens` to your `.cabal` `build-depends` (or `cabal install --lib lens`
for GHCi experiments).

```haskell
{-# LANGUAGE TemplateHaskell #-}
import Control.Lens

data User = User { _name :: String, _age :: Int } deriving Show
makeLenses ''User   -- generates: name :: Lens' User String, age :: Lens' User Int

main :: IO ()
main = do
  let u = User "Tom" 40
  print (u ^. name)                    -- "Tom"
  print (u & age +~ 1)                 -- User {_name = "Tom", _age = 41}
  print (("hi", (1, 2)) ^. _2 . _1)    -- 1  (nested access composes with `.`)
  print ([(1,"a"),(2,"b")] & each . _1 %~ (*10))  -- [(10,"a"),(20,"b")]
```

The underscore-prefix field convention exists so Template Haskell can strip it
to name the generated lenses.

## Architecture / How It Works

A `Lens s t a b` is not a data type. It is the function type
`forall f. Functor f => (a -> f b) -> s -> f t`[^2]. Instantiating `f` at
`Const a` turns the lens into a getter; instantiating at `Identity` turns it
into a setter. `view`, `set`, and `over` are just these instantiations. Because
every optic is literally a function, composition is function composition — no
special operator, no wrapper types, no runtime cost beyond what GHC's inliner
already removes.

The optic hierarchy is expressed by varying the constraint on `f` (and, for
`Prism`/`Iso`, adding a `Profunctor` constraint on the argument): `Traversal`
requires `Applicative`, `Fold` adds `Contravariant`, `Setter` uses a dedicated
`Settable` class. Since constraints only ever get weaker along the hierarchy, a
`Lens` *is a* `Traversal` *is a* `Fold` by plain subsumption — passing one where
the other is expected just works, which is elegant, and which also means type
errors mention `Contravariant f, Applicative f => ...` instead of anything a
newcomer can act on.

Other load-bearing pieces:

- **Template Haskell codegen** — `makeLenses`, `makePrisms`, `makeClassy`,
  `makeFields` generate optics from record definitions. `makeFields` produces
  per-field typeclasses (`HasName`) to survive field-name collisions.
- **`Data.Data.Lens`** — `uniplate`-style generic traversals (`biplate`,
  `template`) driven by the `Data` typeclass; maximally convenient, slow.
- **The Kmett stack** — `lens` sits atop a tower of the author's own packages
  (`profunctors`, `comonad`, `kan-extensions`, `semigroupoids`, `bifunctors`,
  `reflection`, `free`, ...). This is where the dependency weight comes from:
  the abstractions are principled, and you download all of them.

## Production Notes

**Dependency weight is the classic objection.** Depending on `lens` pulls in
dozens of transitive packages and noticeably lengthens cold builds. The
community norm: applications depend on `lens` freely; *libraries* that only
need `^.`/`.~`/`makeLenses` depend on `microlens` instead, which is
API-compatible for the core subset with near-zero dependencies[^4].

**Type errors are the second objection.** Because optics are type synonyms
over rank-2 polymorphic functions, a misused optic produces errors about
functor constraints rather than "you used a Fold where a Lens is needed."
Well-Typed's `optics` library exists specifically to fix this, using opaque
optic types with curated error messages, at the cost of a dedicated
composition operator (`%`) instead of `(.)`[^3].

**API sprawl is real.** Several hundred exported names and a large operator
zoo (`^..`, `^?`, `<>~`, `<+=`, `<<%~`, ...). Teams that adopt `lens`
successfully usually agree on a small sanctioned subset; unrestricted use
makes code review a decoding exercise.

**Lens laws are unchecked.** `makeLenses` output is lawful by construction,
but hand-written optics that violate get-put/put-get laws typecheck fine and
fail subtly under composition. Test hand-rolled optics.

**GHC-version churn, not API churn.** Since 4.0 the API has been broadly
stable; most releases exist to support new GHC versions, and `lens` tracks new
GHC releases promptly. Upgrade pain is generally inherited from GHC, not from
`lens` itself.

**Modern GHC erodes the entry-level use case.** `OverloadedRecordDot` (GHC
9.2+) covers plain nested field *access* without any library. What it does not
cover — polymorphic updates, prisms over sum types, traversals, `Fold`
queries — is where `lens` still earns its weight.

## When to Use / When Not

**Use when:**
- An application works with deeply nested or sum-heavy data (JSON via
  `lens-aeson`, compiler ASTs, game state) and updates dominate.
- You want one vocabulary for get/set/modify/fold across every container,
  including state-monad combinators (`use`, `.=`, `%=`).
- You already pay for the Kmett stack via other dependencies, so the marginal
  cost is near zero.

**Avoid when:**
- You are writing a library and only need basic field access — use
  `microlens` and keep your dependency footprint small.
- The team is new to Haskell: the error messages and operator zoo have a real
  onboarding cost; `optics` or plain record syntax is kinder.
- Simple field reads are all you need — `OverloadedRecordDot` does that with
  zero dependencies.

## Alternatives

- well-typed/optics — use instead when you want the same optic hierarchy with
  opaque types and dramatically better error messages, and can accept `%` for
  composition.
- stevenfontanella/microlens — use instead in libraries: the compatible core
  subset of `lens` with almost no dependencies.
- kcsongor/generic-lens — use instead of (or with) Template Haskell: derives
  optics via `Generic`, works with `DuplicateRecordFields`.
- sebastiaanvisser/fclabels — earlier take on first-class labels; mostly of
  historical interest now.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2012 | Initial Hackage release; rapid 1.x–3.x iteration through 2012–2013[^1]. |
| 4.0 | 2014 | Major consolidation: indexed and plain combinators unified, profunctor-encoded `Prism`/`Iso`. |
| 5.0 | 2021 | GHC 9.0 support; dropped legacy GHC compatibility. |
| 5.2 | 2022 | GHC 9.2/9.4-era maintenance line. |
| 5.3 | 2023 | GHC 9.8-era support; ongoing 5.3.x point releases[^5]. |

## References

[^1]: lens on Hackage — release history and reverse dependencies. https://hackage.haskell.org/package/lens
[^2]: Twan van Laarhoven, "CPS based functional references" — 2009, the encoding `lens` is built on. https://www.twanvl.nl/blog/haskell/cps-functional-references
[^3]: Well-Typed, "Announcing the optics library" — motivation section is an explicit critique of `lens`'s error messages. https://well-typed.com/blog/2019/09/announcing-the-optics-library/
[^4]: microlens on Hackage — "compatible with lens" scope statement. https://hackage.haskell.org/package/microlens
[^5]: lens CHANGELOG. https://github.com/ekmett/lens/blob/master/CHANGELOG.md

## Tags

haskell, optics, lenses, functional-programming, data-access, records, template-haskell, library, traversals, immutability
