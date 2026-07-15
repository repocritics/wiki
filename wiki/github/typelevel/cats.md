# typelevel/cats

> Type-class-based functional programming abstractions for Scala — the foundation the Typelevel ecosystem is built on.

[GitHub repo](https://github.com/typelevel/cats) ·
[Official website](https://typelevel.org/cats/) ·
[License: MIT](https://github.com/typelevel/cats/blob/main/COPYING)

## Overview

Cats is a Scala library that ports the standard functional-programming type classes — `Functor`, `Applicative`, `Monad`, `Traverse`, `Semigroup`, `Monoid`, `Foldable`, and dozens more — into Scala's implicit-based encoding, along with data types (`Validated`, `Ior`, `NonEmptyList`, `Chain`, `EitherT`, `Kleisli`, `Free`) that exploit them[^1]. The name is a shortening of "category," from category theory, though the README is explicit that you do not need category theory to use it. Scala's standard library mixes object-oriented and functional styles; Cats supplies the disciplined, lawful functional layer the stdlib leaves out.

Cats is less an application framework than a *substrate*. Its real significance is that it standardized the abstractions the rest of the Typelevel ecosystem depends on: cats-effect (the `IO` runtime), fs2 (streaming), http4s (server), doobie (JDBC), and circe (JSON) all agree on Cats' type classes, so their APIs compose. This is also the defining tension: adopting Cats is rarely a small decision. A team either commits to the pure-FP idiom — type classes, referential transparency, `IO`-wrapped effects — or it fights the library at every call site.

The other structural fact worth internalizing is the **module split**. The `typelevel/cats` repo is deliberately minimal: kernel, core, laws, free, testkit. Effects, monad transformers (`cats-mtl`), and derivation (`kittens`) live in *separate repositories* with independent release cadences[^1]. Newcomers routinely conflate "Cats" with "cats-effect"; they are different projects with different versioning and different learning curves.

## Getting Started

```scala
// build.sbt — JVM, Scala.js, and Scala Native all supported
libraryDependencies += "org.typelevel" %% "cats-core" % "2.13.0"
```

```scala
import cats.syntax.all._

// Monoid: combine anything with a defined "empty" and "combine"
List(1, 2, 3).combineAll                       // 6
Map("a" -> 1).combine(Map("a" -> 2, "b" -> 5)) // Map(a -> 3, b -> 5)

// Traverse: flip List[Option[A]] into Option[List[A]], short-circuiting
List("1", "2", "3").traverse(_.toIntOption)    // Some(List(1, 2, 3))
List("1", "x", "3").traverse(_.toIntOption)    // None
```

```scala
// Validated: accumulate ALL errors instead of failing on the first
import cats.data.ValidatedNel
import cats.syntax.all._

def check(name: String, age: Int): ValidatedNel[String, (String, Int)] = (
  Validated.condNel(name.nonEmpty, name, "name empty"),
  Validated.condNel(age >= 0, age, "age negative")
).tupled

check("", -1)  // Invalid(NonEmptyList("name empty", "age negative"))
```

## Architecture / How It Works

Cats is a hierarchy of **type classes** encoded as traits with implicit instances. `Monad[F[_]]` extends `FlatMap[F]` and `Applicative[F]`; `Applicative` extends `Apply` and `Functor`; and so on up an inheritance lattice that mirrors the algebraic one. You program against the most general capability that suffices (`Traverse`, not `List`), and the compiler resolves the concrete instance from implicit scope.

The published artifacts layer as follows:

- **cats-kernel** — the smallest algebraic classes (`Eq`, `Order`, `Semigroup`, `Monoid`) with no dependencies. Split out specifically so libraries can depend on it without pulling in all of core.
- **cats-core** — `Functor` through `Monad`, `Traverse`, `Foldable`, the syntax enrichments (`cats.syntax.all._`), and data types.
- **cats-laws** — the type-class laws expressed as ScalaCheck properties.
- **cats-testkit** — harness for running those laws against your own instances. This is the mechanism that makes "lawful" a checkable claim, not a slogan.
- **algebra** / **alleycats-core** — algebraic structures, and instances that are useful but *not* lawful (kept out of core on purpose).

**Binary compatibility is the load-bearing engineering constraint.** After 1.0.0 the project adopted strict SemVer and treats backward binary compatibility between MINOR and PATCH releases as near-inviolable[^2]. This is not pedantry: because Cats sits at the bottom of a deep dependency diamond, a single incompatible bump would fracture every downstream library simultaneously. The consequence is a very conservative core that has stayed on the `2.x` line for years.

Cats cross-publishes for Scala 2.12, 2.13, and Scala 3, and for the JVM, Scala.js, and Scala Native. The Scala 3 build reworks parts of the implicit machinery to use `given`/`using`, but the surface API is kept source-compatible across compilers where possible.

## Production Notes

**Implicit resolution and compile times.** Cats leans hard on implicit search. Deep type-class stacks (`EitherT[OptionT[IO, *], E, A]`) and large `for`-comprehensions can make the Scala 2 compiler slow and its error messages opaque. Scala 3 improved both, but "the implicit isn't found and I can't tell why" remains the archetypal Cats debugging session. Keep type ascriptions explicit at API boundaries.

**Partial unification.** On Scala 2.12 you must enable the SI-2712 fix or `Functor`/`Traverse` over types like `Either[E, *]` silently fail to resolve. Add `scalacOptions += "-Ypartial-unification"`. It is on by default from 2.13 onward, and the flag was *removed* in 2.13 — so a shared `build.sbt` that sets it unconditionally breaks the 2.13 build.[^1]

**`import cats.syntax.all._` vs. granular imports.** The wildcard syntax import is convenient but pulls a large surface into scope and can slow compilation and occasionally cause ambiguous-implicit clashes with other libraries (notably Scalaz-in-the-same-file, or older Scala collection syntax). Granular imports (`cats.syntax.option._`) are the escape hatch.

**Cats vs. cats-effect versioning.** These move independently. cats-effect 3 is a full rewrite versus cats-effect 2 with an incompatible runtime model; a project's cats-core version and its cats-effect version are chosen separately, and mismatched *ecosystem* libraries (one built against CE2, another against CE3) will not link. Check the effect-library matrix before upgrading, not just cats-core.

**The idiom is all-or-nothing.** Cats rewards codebases that go pure-FP throughout and punishes half-measures. Sprinkling `IO` and `Validated` into an otherwise imperative Scala app tends to produce the worst of both: the ceremony of FP without the compositional payoff. Budget for team ramp-up; the learning curve is real and front-loaded.

## When to Use / When Not

**Use when:**
- You are already in — or committing to — the Typelevel stack (cats-effect, fs2, http4s, doobie); Cats is the non-optional base layer.
- You want lawful, testable abstractions (`Monoid`, `Traverse`, `Eq`) and error accumulation (`Validated`) that the stdlib lacks.
- You value long-term binary stability in a foundational dependency.

**Avoid when:**
- The team is new to Scala or FP and the project is on a tight deadline — the abstraction tax lands before the payoff.
- You want a small, mostly-imperative Scala service; the stdlib plus a light effect wrapper may be enough.
- You are on the ZIO stack, whose runtime and design philosophy are a competing, largely parallel universe (interop exists but adds friction).

## Alternatives

- scalaz/scalaz — the original Scala FP library that informed Cats' design; broader, older, less binary-compat discipline. Use it if you are already invested or need its specific data types.
- zio/zio — effect-system-first alternative with its own type classes and runtime; use ZIO when you want an integrated batteries-included effect platform rather than a type-class substrate.
- typelevel/cats-effect — not a competitor but the effect layer *on top of* Cats; you almost always add it when you need `IO`.
- Scala standard library — for simple programs, `Option`/`Either`/`for` cover a surprising amount without any type-class ceremony.
- scala/scala3 (contextual abstractions) — Scala 3's `given`/`using` and opaque types reduce some of the boilerplate Cats historically compensated for.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2015-08 | First public release under typelevel[^1]. |
| 1.0.0 | 2018-01 | First stable release; SemVer + binary-compatibility commitment adopted[^2]. |
| 2.0.0 | 2019-09 | Scala 2.13 support; effects already split into cats-effect repo. |
| 2.6.x | 2021 | Scala 3 cross-publishing lands in the 2.x line. |
| 2.9.0 | 2022-12 | Version pinned in the repo README at time of writing. |
| 2.13.0 | 2025 | Latest 2.x line; copyright header spans 2015–2025. |

## References

[^1]: typelevel/cats README — overview, modules, versioning policy, and the `-Ypartial-unification` note. https://github.com/typelevel/cats
[^2]: Cats binary-compatibility and versioning decision, issue #1233. https://github.com/typelevel/cats/issues/1233

## Tags

scala, functional-programming, type-classes, category-theory, jvm, scala-js, scala-native, typelevel, monad, library, fp
