# zio/zio

> A type-safe, composable effect system for asynchronous and concurrent programming in Scala, built on lightweight fibers.

[GitHub repo](https://github.com/zio/zio) ·
[Official website](https://zio.dev) ·
[License: Apache-2.0](https://github.com/zio/zio/blob/series/2.x/LICENSE)

## Overview

ZIO is a Scala library for building concurrent, asynchronous, and resource-safe programs around a single data type, `ZIO[R, E, A]`: an effect that requires an environment `R`, may fail with a typed error `E`, or succeed with a value `A`[^1]. Effects are values — pure descriptions of what should happen — and nothing runs until a runtime executes them at the program's edge. This is the same "functional effects" idea as Haskell's `IO`, extended with a typed error channel and a dependency channel. It began around 2017 as "Scalaz ZIO" under John A. De Goes, went standalone, reached 1.0 in August 2020, and had a substantial runtime and API rewrite in 2.0 (June 2022)[^2].

The defining tension is **the effect-system tradeoff itself**: you get compile-time-tracked errors, structured concurrency, deterministic resource cleanup, and testable-by-construction code — in exchange for writing all effectful logic in a monadic (`for`-comprehension) style that is not how most Scala or JVM code is written. The whole program divides into "pure ZIO world" and "the outside," and the boundary has real ergonomic cost: interop with `Future`, blocking Java APIs, and callback-based libraries all need explicit bridging. ZIO's second defining trait is its **first-party ecosystem** (zio-http, zio-streams, zio-json, zio-config, zio-kafka, zio-prelude, and more) — a deliberate contrast to the more à-la-carte typelevel stack.

ZIO's direct rival is Cats Effect. The two are technically close (both fiber-based, both `IO`-shaped) but philosophically different: ZIO ships a batteries-included concrete type with typed errors and built-in DI; Cats Effect is a minimal runtime designed to sit under the Cats typeclass hierarchy in tagless-final style. Choosing between them is often a team-culture decision, not a technical one.

## Getting Started

```scala
// build.sbt — cross-published for Scala 2.12/2.13/3, also Scala.js and Scala Native
libraryDependencies += "dev.zio" %% "zio" % "2.1.14"  // check Maven Central for latest 2.1.x
```

```scala
import zio._

object Main extends ZIOAppDefault {
  val run =
    for {
      _    <- Console.printLine("What's your name?")
      name <- Console.readLine
      _    <- Console.printLine(s"Hello, $name!")
    } yield ()
}
```

Concurrency is fiber-based; `fork` spawns a fiber, `join` awaits it, and `ZIO.foreachPar` runs a collection in parallel with automatic interruption on failure:

```scala
for {
  fiber <- longRunningTask.fork      // returns immediately, runs concurrently
  _     <- otherWork
  result <- fiber.join               // await; propagates errors and interruption
} yield result
```

## Architecture / How It Works

**The type.** `ZIO[R, E, A]` is the whole API surface. Common aliases: `Task[A]` = `ZIO[Any, Throwable, A]`, `UIO[A]` = `ZIO[Any, Nothing, A]` (cannot fail), `IO[E, A]` = `ZIO[Any, E, A]`, `RIO[R, A]`, `URIO[R, A]`. The `Nothing` error type is load-bearing: it lets the compiler prove an effect is infallible.

**Fibers.** ZIO runs effects on green threads called fibers, multiplexed M:N onto a small JVM thread pool. Fibers are cheap (millions are feasible), never block the underlying OS thread when they suspend, and are fully interruptible. Structured concurrency means a fiber's children are supervised and cancelled with it. The 2.0 rewrite replaced the earlier executor with a new fiber runtime that is stack-safe and reduced allocation overhead[^2].

**Error model.** Three distinct failure kinds: *failures* (the typed `E` channel, expected domain errors), *defects* (unexpected `Throwable`s outside `E`), and *interruptions*. All are unified in the `Cause[E]` data type, which can hold multiple parallel failures plus defects — so a `foreachPar` that fails in two fibers preserves both. Confusing failures with defects is the most common conceptual error for newcomers.

**Environment / DI.** The `R` parameter carries dependencies. In ZIO 1.x this used a `Has[_]` map machinery; **2.0 removed `Has`** and made `R` a plain intersection type, wiring services with `ZLayer` and `provideLayer`. `ZLayer` is a memoized, parallelized, resource-safe dependency graph — effectively compile-time-checked DI.

**Resources.** `ZIO.acquireRelease` plus `Scope` guarantee release even under failure or interruption. `Scope` replaced the older `ZManaged` type in 2.0, folding resource management back into the core `ZIO` type.

**Beyond the core.** `ZStream`/`ZChannel`/`ZSink`/`ZPipeline` provide pull-based, back-pressured streaming; `STM`/`TRef` provide composable software transactional memory for lock-free concurrency; `ZIO Test` is a first-party test framework whose specs are themselves effects, with `TestClock` for deterministic time. All of these are in-repo or in tightly-versioned sibling repos under the `zio` org.

## Production Notes

**The interop boundary is where time goes.** Wrapping blocking Java calls needs `ZIO.attemptBlocking` (routes to a dedicated blocking pool so you don't starve the main fiber executor). `Future` interop needs `ZIO.fromFuture`. Callback APIs need `ZIO.async`. Forgetting `attemptBlocking` around JDBC/file/socket calls is a classic production footgun: it silently pins fiber-runtime threads and throttles the whole app under load.

**Fiber dumps, not stack traces.** Because logical flow spans fibers, JVM stack traces are near-useless. ZIO maintains its own execution traces and fiber dumps; learn to read them. Async traces add some overhead but are on by default because debugging without them is impractical.

**1.x → 2.0 was a hard migration.** The removal of `Has`, replacement of `ZManaged` with `Scope`, renamed combinators, and changed layer syntax mean 1.x code does not compile on 2.x without work. There is an official `zio-2.x` Scalafix migration but expect manual cleanup, especially around layers and managed resources. Third-party ecosystem libraries updated on a lag.

**Compile times and inference.** Heavy `for`-comprehensions with many layers can stress the Scala compiler and produce famously long error messages when a single service is missing from `R`. `ZLayer.make`/`provide` will tell you exactly which dependency is unsatisfied, but the message can be a wall of types. Scala 3 improves some of this; Scala 2.12 users see the worst of it.

**Ecosystem version coupling.** Staying on the `dev.zio` stack (zio-http, zio-json, zio-kafka, etc.) means tracking compatible versions together; a ZIO core bump often waits on the satellite libraries. zio-http in particular has had API churn across its own pre-1.0 history.

**Cats Effect interop is real but not free.** `zio-interop-cats` lets you run Cats Effect / fs2 / http4s libraries on the ZIO runtime, which many teams rely on. It works, but you inherit two mental models and occasional typeclass-instance friction at the seam.

## When to Use / When Not

**Use when:**
- You want typed, tracked errors and are willing to write effectful code monadically.
- You need high-concurrency (many thousands of in-flight operations) with structured cancellation and guaranteed resource cleanup.
- You value a cohesive first-party stack (HTTP, JSON, config, streams, Kafka) with consistent design.
- Deterministic, dependency-injected, fast tests matter to you (`ZIO Test` + `TestClock`).

**Avoid when:**
- The team is new to functional Scala and the project timeline can't absorb the learning curve.
- Your codebase is mostly synchronous CRUD glue — the effect-system overhead buys little.
- You're committed to the tagless-final / Cats typeclass ecosystem; Cats Effect fits that culture better.
- You want direct-style code on JDK 21+ virtual threads without a monadic wrapper — a newer generation of libraries targets exactly that.

## Alternatives

- typelevel/cats-effect — the other major Scala effect runtime; use it when you want a minimal `IO` under the Cats typeclass ecosystem and tagless-final abstractions rather than one concrete batteries-included type.
- monix/monix — earlier `Task`/`Observable` library; use it mainly to maintain existing Monix code, as momentum has moved to ZIO and Cats Effect.
- apache/pekko (the Apache-licensed Akka fork) — use when you want the actor model and distributed messaging instead of an effect monad; note Akka itself moved to the BSL license, prompting Pekko.
- softwaremill/ox — use when you want direct-style concurrency on JDK virtual threads (Project Loom) without monadic effects.
- getkyo/kyo — use when you want an algebraic-effects approach that composes effects more granularly than a single `ZIO` type.

## History

| Version | Date | Notes |
|---------|------|-------|
| Scalaz ZIO | 2017 | Origins as an effect type in the Scalaz orbit, led by John De Goes. |
| 1.0.0 | 2020-08 | First stable release; typed errors, fibers, `ZManaged`, `Has`-based environment[^1]. |
| 2.0.0 | 2022-06 | Runtime rewrite; removed `Has`, replaced `ZManaged` with `Scope`, new fiber runtime[^2]. |
| 2.1.0 | 2024 | Continued 2.x line; performance and API refinements (see release notes for specifics). |

## References

[^1]: ZIO documentation — "Getting Started" and the `ZIO` data type overview. https://zio.dev/reference/core/zio/
[^2]: ZIO 2.0 release announcement and migration guide. https://zio.dev/guides/migrate/zio-2.x-migration-guide/

## Tags

scala, jvm, functional-programming, effect-system, concurrency, asynchronous, fibers, streams, software-transactional-memory, dependency-injection, resource-safety
