# typelevel/fs2

> Compositional, purely functional streaming I/O for Scala — the plumbing layer under http4s, doobie, skunk, and fs2-kafka.

[GitHub repo](https://github.com/typelevel/fs2) ·
[Official website](https://fs2.io) ·
[License: MIT](https://github.com/typelevel/fs2/blob/main/LICENSE) (GitHub reports `NOASSERTION` because the file appends attribution notices for code derived from scodec; the operative text is MIT[^1])

## Overview

FS2 ("Functional Streams for Scala") is a streaming library built on Cats and Cats Effect. Its core abstraction, `Stream[F, O]`, is a lazy *description* of a process that may evaluate effects in `F` and emit values of type `O` — nothing runs until the stream is compiled to an effect and that effect is executed. The project's stated design goals are compositionality, expressiveness, resource safety, and speed[^2]. It began life in 2013 as scalaz-stream by Paul Chiusano, the companion library to *Functional Programming in Scala*, and was renamed FS2 for the 0.9 rewrite before migrating from scalaz to Cats[^3].

FS2's defining tradeoff is that it buys guaranteed resource safety and composable concurrency at the price of a demanding mental model. Files, sockets, and processes opened inside a stream are released exactly once, even under errors, interruption, or fiber cancellation — but to get there you accept effect-polymorphic signatures (`Stream[F, O]` with `Async[F]` constraints), the `Pull` monad for custom operators, and stack traces that point at the interpreter rather than your code.

At ~2.4k stars it looks mid-sized, but the star count understates its footprint: it is the substrate of much of the Typelevel ecosystem — http4s models HTTP bodies as FS2 streams, doobie streams JDBC result sets, skunk speaks the Postgres wire protocol over fs2-io, and fs2-kafka wraps the Kafka client[^4]. If you run Typelevel-stack services, you run FS2 whether you write `Stream` or not. The project cross-builds for Scala 2.12, 2.13, and 3 on JVM, Scala.js, and Scala Native[^2], and remains actively maintained (pushed July 2026; v3.13.0 released March 2026[^5]).

## Getting Started

```scala
// build.sbt
libraryDependencies ++= Seq(
  "co.fs2" %% "fs2-core" % "3.13.0",
  "co.fs2" %% "fs2-io"   % "3.13.0"  // files, TCP/UDP, TLS, processes
)
```

```scala
import cats.effect.{IO, IOApp}
import fs2.{Stream, text}
import fs2.io.file.{Files, Path}

object Convert extends IOApp.Simple {
  def fahrenheitToCelsius(f: Double): Double = (f - 32.0) * (5.0 / 9.0)

  val converter: Stream[IO, Unit] =
    Files[IO].readAll(Path("fahrenheit.txt"))
      .through(text.utf8.decode)
      .through(text.lines)
      .filter(s => s.trim.nonEmpty && !s.startsWith("//"))
      .map(line => fahrenheitToCelsius(line.toDouble).toString)
      .intersperse("\n")
      .through(text.utf8.encode)
      .through(Files[IO].writeAll(Path("celsius.txt")))

  def run: IO[Unit] = converter.compile.drain
}
```

Constant memory regardless of file size; both files are closed on completion, error, or cancellation — no `try/finally` in sight.

## Architecture / How It Works

- **`Stream[F, O]` is a program, not a collection.** Operators (`map`, `flatMap`, `merge`, `concurrently`, `through`) build up a description. `.compile.drain` / `.compile.toList` / `.compile.fold` interpret it into a single `F[A]`, which Cats Effect then runs.
- **Chunks are the unit of execution.** Streams internally carry `Chunk[O]` (array-backed, immutable) rather than single elements, and chunk-aware operators amortize interpreter overhead across whole chunks. This is where most of FS2's throughput comes from — and losing chunkiness is where most of it goes (see Production Notes).
- **`Pull[F, O, R]` is the low-level API.** A `Pull` reads from streams, emits output, and terminates with a result `R`; custom stateful operators (windowing, batching, parsers) are written by destructuring a stream with `stream.pull.uncons` and rebuilding output. It is a free-monad-style structure the compile loop interprets.
- **Scopes track resources.** `Stream.resource` / `Stream.bracket` register finalizers in a hierarchical scope tree; scopes close deterministically when a stream section completes, fails, or is interrupted. Interruption itself is built on Cats Effect fiber cancellation, so `stream.interruptWhen(signal)` composes with the rest of the CE runtime.
- **Effect polymorphism.** `fs2-core` is written against Cats Effect typeclasses (`Concurrent`, `Temporal`, `Async`), not `IO` directly, so libraries built on FS2 stay runtime-agnostic. Concurrency primitives (`Topic`, `Signal`, `Channel`) live in FS2; `Queue` and friends moved down into cats-effect-std with CE3[^6].
- **`fs2-io`** provides `Files`, TCP/UDP `Network`, TLS, and process spawning on all three platforms — Node.js APIs back the Scala.js build, which is how http4s serverless deployments on Node work.

## Production Notes

**Chunk-size collapse is the classic perf bug.** `evalMap` processes one element per effect and yields singleton chunks downstream; a pipeline that was moving thousand-element chunks suddenly does per-element interpreter work. `evalMapChunk`, `mapChunks`, or restructuring around `chunkN` restores throughput. Profile chunk sizes before blaming FS2 itself.

**Per-element overhead is real.** Against raw loops or Akka/Pekko Streams' fused stages, FS2 pays for purity and safety; for network- or DB-bound services this is noise, but it is a poor fit for tight numeric inner loops.

**Debugging is unpleasant.** Laziness means exceptions surface at `.compile` with interpreter frames, far from the operator that failed. Cats Effect tracing helps; discipline about small named pipes helps more.

**Interruption happens at chunk/effect boundaries, not preemptively.** A long synchronous computation inside `map` cannot be interrupted mid-flight.

**Backpressure needs bound awareness.** `groupWithin`, `Topic` fan-out, and merge combinators are only as safe as their buffers; an unbounded `Channel` or a slow `Topic` subscriber is the standard slow-leak incident in FS2 services.

**Upgrade history matters.** The 2.x → 3.0 jump rode the Cats Effect 2 → 3 migration (new typeclass hierarchy, `Blocker` removal, rewritten fs2-io API) and was a genuine porting effort for large codebases[^7]. Within the 3.x line, binary compatibility is checked with MiMa and upgrades have been routine.

**Team cost.** Effective use presumes fluency with Cats Effect. Onboarding for a `Pull`-level codebase is measurably harder than for a framework-style stack; most teams stay at the combinator level and are fine.

## When to Use / When Not

**Use when:**
- You are already on the Typelevel stack (http4s, doobie, skunk) — FS2 is the native way to stream request bodies, query results, and socket traffic.
- You need constant-memory processing of unbounded/large data with guaranteed resource cleanup under failure and cancellation.
- You want structured concurrency over streams — `concurrently`, `parEvalMap`, `merge`, `Topic` fan-out — with cancellation that composes.
- You target Scala.js or Scala Native and want the same streaming API there.

**Avoid when:**
- Your team is not invested in Cats Effect; the abstraction stack (typeclasses, `F[_]`, `Pull`) dominates the learning budget.
- You need distributed streaming with state, partitioning, and delivery guarantees — that is Kafka Streams / Flink territory, not an in-process streaming library.
- You are in Akka/Pekko land already; mixing two streaming runtimes in one service multiplies confusion for little gain.
- A simple collection pipeline or iterator suffices — FS2 on `List`-sized data is ceremony without payoff.

## Alternatives

- zio/zio — ZStream offers comparable in-process streaming inside the ZIO ecosystem; use it when your codebase is ZIO- rather than Cats-based.
- apache/pekko — Pekko Streams (open-source Akka fork) when you want fused push-based stages, Reactive Streams interop, and an actor ecosystem.
- akka/akka — the original, now under Lightbend's BSL license; only when you are a paying Akka customer.
- reactor/reactor-core — JVM Reactive Streams for Java/Spring shops; use it when Scala and purity are not requirements.
- monix/monix — `Observable`-style push streaming for Scala; historically significant, but development has largely stalled — prefer FS2 or ZStream for new code.

## History

| Version | Date | Notes |
|---------|------|-------|
| scalaz-stream 0.1 | 2013 | Initial release by Paul Chiusano; repo created 2013-03[^3]. |
| 0.9 | 2016 | Rewrite and rename to FS2; new core, chunking model[^3]. |
| 0.10 | 2018-01 | Cats/cats-effect migration completed; scalaz dependency gone[^5]. |
| 1.0 | 2018-10 | First stable FS2; cats-effect 1.0 alignment[^5]. |
| 2.0 | 2019-09 | `Segment` removed in favor of plain `Chunk`; simpler, faster core[^5]. |
| 3.0 | 2021-03 | Cats Effect 3, rewritten fs2-io, Scala 3 support[^7]. |
| 3.13.0 | 2026-03 | Current stable in the long-running binary-compatible 3.x line[^5]. |

## References

[^1]: FS2 LICENSE — MIT, Copyright (c) 2013 Paul Chiusano and contributors, with appended scodec-derived-code notices. https://github.com/typelevel/fs2/blob/main/LICENSE
[^2]: FS2 README — design goals, platform support, Cats/Cats-Effect foundation. https://github.com/typelevel/fs2#overview
[^3]: FS2 documentation page — history and talks on the scalaz-stream → FS2 lineage. https://fs2.io/#/documentation
[^4]: FS2 ecosystem page — libraries built on FS2 (http4s, doobie, skunk, fs2-kafka, and others). https://fs2.io/#/ecosystem
[^5]: FS2 GitHub releases (v0.10.0 2018-01-31, v1.0.0 2018-10-05, v2.0.0 2019-09-10, v3.13.0 2026-03-12). https://github.com/typelevel/fs2/releases
[^6]: Cats Effect 3 `std` — `Queue`, `Semaphore`, and friends moved out of FS2. https://typelevel.org/cats-effect/docs/std/queue
[^7]: FS2 v3.0.0 release notes — Cats Effect 3 migration, fs2-io rework. https://github.com/typelevel/fs2/releases/tag/v3.0.0

## Tags

scala, streaming, functional-programming, cats-effect, typelevel, io, concurrency, resource-safety, scala-js, scala-native, library
