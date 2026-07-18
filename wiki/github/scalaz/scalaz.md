# scalaz/scalaz

> The original principled functional programming library for Scala — type classes, purely functional data structures, and the project whose community schisms produced Cats and ZIO.

[GitHub repo](https://github.com/scalaz/scalaz) ·
[Scaladoc](https://scalaz.github.io/scalaz/) ·
[License: BSD-style](https://github.com/scalaz/scalaz/blob/master/LICENSE.txt)

## Overview

Scalaz is a library that ports the Haskell-style typed functional programming vocabulary to Scala: a type class hierarchy (`Functor`, `Applicative`, `Monad`, `Traverse`, `Monoid`, …), instances for standard library and Scalaz-native types, and purely functional data structures the stdlib lacks or gets wrong (`NonEmptyList`, `Validation`, the `\/` disjunction, `IList`, `ISet`, monad transformers)[^1]. Started by Tony Morris around 2008 and on GitHub since January 2010, it predates every other Scala FP library and defined the idioms — context bounds for type classes, `scalaz.syntax` extension-method imports, instance orphan management via `scalaz.std` — that its successors inherited.

The honest framing in 2026: Scalaz is the ancestor, not the mainstream. Community conflicts in the mid-2010s led Typelevel to build Cats as a friendlier, more modular reimplementation of the same ideas[^2], and in 2018–2019 the Scalaz 8 effect-system effort (`scalaz-zio`) split off to become ZIO, an independent project that now dwarfs its parent[^3]. Scalaz 8 itself was never released. What remains is the 7.x line — stable, conservatively maintained (largely by Kenji Yoshida), cross-built against Scala 2.12, 2.13, 3.x, Scala.js, and Scala Native[^1]. The repo still receives regular commits (last push July 2026) but they are overwhelmingly cross-version maintenance and dependency updates, not new design. 4.7k stars and 157 open issues reflect a mature library with a small, loyal user base — mostly long-lived codebases that adopted it before Cats existed.

The defining tradeoff: Scalaz's type class encoding is coherent and battle-tested, but it is a parallel universe. Its type classes are incompatible with Cats' (interop requires shim libraries), and virtually all modern Scala FP ecosystem libraries (http4s, doobie, fs2, circe) target Cats. Choosing Scalaz today means choosing its data structures on their own merits while giving up the surrounding ecosystem.

## Getting Started

sbt:

```scala
libraryDependencies += "org.scalaz" %% "scalaz-core" % "7.3.9"
```

Type class usage, instance imports a-la-carte[^1]:

```scala
import scalaz._
import std.option._, std.list._  // instances for Option and List

Apply[Option].apply2(some(1), some(2))((a, b) => a + b)
// res0: Option[Int] = Some(3)

Traverse[List].traverse(List(1, 2, 3))(i => some(i))
// res1: Option[List[Int]] = Some(List(1, 2, 3))
```

Or the all-in import for Scalaz 6-style ergonomics:

```scala
import scalaz._, Scalaz._

List(some(1), none[Int]).suml   // Some(1)
NonEmptyList(1, 2, 3).cojoin    // comonadic duplicate
```

## Architecture / How It Works

Scalaz 7 (2013) was a ground-up reorganization of the library around implicit-scope discipline, and its structure has been stable since[^4]:

- **Type class hierarchy** — traits forming a subtyping lattice (`Monad[F] <: Applicative[F] <: Apply[F] <: Functor[F]`), so a `Monad` instance satisfies a `Functor` context bound directly. Derived combinators live on the type class itself, so instances are usable standalone without syntax imports.
- **`scalaz.std`** — instances for stdlib/Java types live here, not in type class companions, and must be imported (`import scalaz.std.option._`). One implicit value provides multiple type classes (e.g. `Traverse[Option] with MonadPlus[Option]`), which sidesteps the ambiguous-implicit bugs that plagued Scalaz 6's "constructive implicits."
- **`scalaz.syntax`** — extension methods (`fa.map`, `oi.join`, `x.some`) are strictly segregated from the core and importable per type class, to limit implicit search space and IDE autocomplete pollution. Everything is callable without syntax.
- **Transformer-first data types** — `State` is a type alias for `StateT[Id, S, A]` with `type Id[A] = A`; capabilities of `OptionT[F, _]`, `EitherT`, `StateT` scale with `F` via a hierarchy of prioritized implicit definitions (subclass-defined implicits win ambiguity resolution). This prioritization machinery is clever and fragile — it is where most confusing implicit-resolution errors originate.
- **Modules** — `scalaz-core` (the bulk), `scalaz-effect` (IO), `scalaz-iteratee` (legacy streaming; superseded in practice by fs2/ZIO Streams everywhere).

Scalaz also ships its own strict/invariant collections (`IList`, `ISet`, `IMap`, `Maybe`, `EphemeralStream`) as law-abiding alternatives to stdlib `List`/`Option` — motivated by stdlib types having invariance and totality issues (e.g. `Option.get`). These are the parts long-time users defend most strongly.

## Production Notes

- **Ecosystem isolation is the dominant cost.** Modern Scala libraries target Cats type classes. Mixing requires interop shims and constant conversion at boundaries; most teams that needed http4s/doobie/circe migrated off Scalaz entirely during 2017–2020. Audit your dependency wishlist before committing.
- **Compile times and implicit resolution.** Heavy type class + syntax usage stresses Scala 2 implicit search; the a-la-carte imports exist partly to mitigate this. Errors from the prioritized transformer-instance hierarchy ("ambiguous implicit values" or a silently weaker instance selected) are hard to diagnose without knowing the encoding.
- **Binary compatibility.** 7.2.x → 7.3.x broke binary and some source compatibility (e.g. data structure reorganizations); within a patch series compatibility is maintained. Pinning mixed 7.2/7.3 transitive dependencies causes runtime `NoSuchMethodError`s — evict carefully.
- **Maintenance profile.** Activity is real but conservative: Scala version cross-building, bug fixes, no roadmap toward a Scalaz 8-style redesign (that energy left with ZIO). Treat it as a finished library, which is acceptable — the type class laws don't change — but do not expect new abstractions or performance work.
- **`scalaz-effect` is not a modern effect system.** Its `IO` lacks the fiber-based concurrency, interruption, and resource-safety machinery of cats-effect or ZIO. Teams wanting a production effect runtime should use those even if they keep scalaz-core for data structures.
- **Learning materials are dated.** The canonical tutorials ("Learning Scalaz", "Towards Scalaz") are from 2013–2015; they remain accurate for 7.x but reference old Scala versions and pre-Cats context.

## When to Use / When Not

**Use when:**
- You maintain an existing Scalaz codebase — 7.3.x is a stable, supported target across Scala 2.12/2.13/3.
- You want `Validation`, `\/`, `NonEmptyList`, `IList` and law-abiding type classes with minimal dependencies and no interest in the wider ecosystem.
- You are studying the design history of type class encodings in Scala — Scalaz 7's README and source remain one of the best documented encodings.

**Avoid when:**
- You are starting a new Scala FP project — Cats + cats-effect is where the libraries, hiring pool, and documentation are.
- You need a concurrent effect runtime — ZIO or cats-effect, not scalaz-effect.
- Your team is new to FP — Scalaz's Haskell-derived naming (`Cobind`, `Zap`, `Unzip`) and dated tutorials make the learning curve steeper than Cats' for no practical gain today.

## Alternatives

- typelevel/cats — use instead for any new project; same abstractions, modular, and the entire modern ecosystem (http4s, doobie, circe, fs2) builds on it.
- zio/zio — use instead when the primary need is a concurrent effect system with built-in dependency injection and streaming; it began as scalaz-zio[^3].
- typelevel/cats-effect — use instead of scalaz-effect for a production IO monad with fibers, interruption, and Resource.
- typelevel/scalacheck — complement, not replacement: property testing used by Scalaz itself to verify type class laws.

## History

| Version | Date | Notes |
|---------|------|-------|
| pre-6 | 2008–2010 | Tony Morris's original library; GitHub repo created 2010-01[^5]. |
| 6.0 | 2011 | `Identity`/`MA`/`MAB` pimp encoding; ergonomic but implicit-ambiguity-prone. |
| 7.0 | 2013 | Full reorganization: `scalaz.std` instances, `scalaz.syntax`, modularization[^4]. |
| 7.2.0 | 2016 | Long-lived stable series; era of peak adoption, and of the Cats exodus. |
| scalaz-zio | 2018 | Scalaz 8's effect system spun out; became ZIO in 2019[^3]. Scalaz 8 never shipped. |
| 7.3.0 | 2020 | Current series; Scala 2.13 and later Scala 3, Scala.js, Scala Native cross-builds. |
| 7.3.9 | current | Stable release per README; maintenance cadence continues as of July 2026[^1]. |

## References

[^1]: Scalaz README — installation, quick start, supported Scala versions. https://github.com/scalaz/scalaz#readme
[^2]: Adelbert Chang, "Towards Scalaz (Part 1)" — Typelevel blog, 2013-10-13; Typelevel's engagement with and eventual reimplementation of Scalaz's ideas as Cats. https://typelevel.org/blog/2013/10/13/towards-scalaz-1.html
[^3]: ZIO — formerly scalaz-zio, originated in the Scalaz 8 effect-system work. https://github.com/zio/zio
[^4]: Scalaz README, "Changes in Version 7" — the design rationale for the 7.x encoding. https://github.com/scalaz/scalaz#changes-in-version-7
[^5]: GitHub repository metadata: created 2010-01-16. https://github.com/scalaz/scalaz

## Tags

scala, functional-programming, type-classes, category-theory, monads, immutable-data-structures, library, scala-js, scala-native, jvm
