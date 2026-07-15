# http4s/http4s

> A minimal, idiomatic Scala interface for HTTP, built on cats-effect and fs2 — servers and clients as pure functions over streaming effects.

[GitHub repo](https://github.com/http4s/http4s) ·
[Official website](https://http4s.org/) ·
[License: Apache-2.0](https://github.com/http4s/http4s/blob/series/0.23/LICENSE)

## Overview

http4s is a Typelevel project that models HTTP as ordinary functions in a purely functional Scala style. A service is a function from `Request` to a (possibly absent) `Response`, both parameterized over an effect type `F[_]`; request and response bodies are `fs2.Stream[F, Byte]`. The README frames it as "Scala's answer to Ruby's Rack, Python's WSGI, Haskell's WAI, and Java's Servlets"[^1] — a low-level, composable substrate rather than a batteries-included web framework.

The defining commitment is effect polymorphism via the cats-effect typeclass hierarchy. You don't write against a concrete runtime; you write against `F[_]: Async` (or a weaker constraint) and choose the concrete effect — typically `cats.effect.IO` — at the edge. Everything is a value: routes compose with `<+>`, middleware is a function `HttpApp[F] => HttpApp[F]`, and there is no reflection, no annotations, and no dependency-injection container. The cost is that the whole ecosystem (cats, cats-effect, fs2) and its concepts (`Kleisli`, `Resource`, `Sync`/`Async`, tagless-final) are prerequisites, not optional.

http4s is a library, not a framework: it deliberately omits templating, ORM, config, and DI, expecting you to assemble those from the Typelevel ecosystem (circe for JSON, doobie/skunk for DB, ciris for config). It underpins higher-level tools — notably tapir, which can derive an http4s server from endpoint descriptions.

## Getting Started

```scala
// build.sbt — ember is the recommended backend as of the 0.23 series
libraryDependencies ++= Seq(
  "org.http4s" %% "http4s-ember-server" % "0.23.36",
  "org.http4s" %% "http4s-dsl"          % "0.23.36",
)
```

```scala
import cats.effect._
import org.http4s._
import org.http4s.dsl.io._
import org.http4s.ember.server.EmberServerBuilder
import com.comcast.ip4s._

object Main extends IOApp.Simple {
  val routes = HttpRoutes.of[IO] {
    case GET -> Root / "hello" / name =>
      Ok(s"Hello, $name.")
  }

  def run: IO[Unit] =
    EmberServerBuilder
      .default[IO]
      .withHost(ipv4"0.0.0.0")
      .withPort(port"8080")
      .withHttpApp(routes.orNotFound)   // 404 for unmatched routes
      .build
      .use(_ => IO.never)               // Resource keeps the server alive
}
```

## Architecture / How It Works

The core types are thin aliases over cats data types, which is why composition "just works" if you know cats:

- `HttpRoutes[F] = Kleisli[OptionT[F, *], Request[F], Response[F]]` — a partial function; `OptionT` encodes "route did not match" so routes can be combined with `<+>` (SemigroupK).
- `HttpApp[F] = Kleisli[F, Request[F], Response[F]]` — a total function; `.orNotFound` turns routes into an app.
- Bodies are `fs2.Stream[F, Byte]`, so streaming, backpressure, and resource safety come from fs2 rather than a bespoke mechanism.
- `EntityDecoder[F, A]` / `EntityEncoder[F, A]` are the implicit codec typeclasses; JSON support (`http4s-circe`) provides instances via `jsonOf[F, A]` / `jsonEncoderOf`.

**Backends are pluggable and this is the ecosystem's live story.** Two matter today:

1. **Ember** — a pure-fp server and client implemented on top of cats-effect + fs2 sockets. It is cross-platform (JVM, Scala.js, Scala Native) and is the recommended default in the 0.23 series. It prioritizes correctness and portability over raw throughput.
2. **Blaze** — the original NIO-based backend. It was extracted to a separate repository (`http4s/blaze`) and is effectively in maintenance mode; new projects are steered to Ember. Blaze historically had higher benchmark throughput.

Additional backends exist via community modules (Netty, Servlet, JDK http-client, async-http-client). Middleware — CORS, GZip, logging, auth, retry, metrics — ships as ordinary functions you wrap around the app.

The `IO`/`Resource`/`Async` model means server lifecycle, connection pools, and clients are `Resource`s: acquisition and release are guaranteed by the runtime, and a leaked connection pool is a compile-visible `Resource` you forgot to `use`.

## Production Notes

**1.0 has not shipped.** The stable, recommended line is the **0.23** series (cats-effect 3, fs2 3); `1.0.0` has been in milestone releases for years and is still at a milestone tag (M47 at time of writing)[^2]. Treat 1.0 milestones as preview: APIs still move. Most production users run 0.23.

**Version/series coupling is the main upgrade hazard.** http4s series are tied to specific cats-effect and fs2 major versions. The 0.21/0.22 → 0.23 jump was the cats-effect 2 → 3 migration, which was pervasive (the effect runtime, `Blocker` removal, `Resource` changes) and could not be done incrementally — your whole dependency graph (circe integration, doobie, fs2 modules) had to cross at once. Mixing a cats-effect 2 and cats-effect 3 library in one build does not link.

**Compile times and error messages.** Being implicit- and typeclass-heavy, http4s inherits Scala's slow compilation and its worst diagnostics: a missing `EntityDecoder`/`EntityEncoder` instance, or an `F` that doesn't satisfy the required `Async` constraint, surfaces as a long "could not find implicit" error far from the real cause. On Scala 2.12 you must enable partial unification (`-Ypartial-unification`); this is default from 2.13 on[^1].

**Backend choice has real consequences.** Ember trades throughput for portability and purity; latency-sensitive, high-QPS services on the JVM have historically gotten more out of Blaze or a Netty-based backend. Benchmark your workload rather than assuming the default is fastest. Blaze's maintenance status is the counter-pressure: you may be optimizing on a backend that is no longer the project's focus.

**Error handling.** Decode failures raise `MessageFailure`, which http4s renders as 4xx by default; unhandled effect errors become 500s. Production services generally install an error-handling middleware (or `HttpApp` wrapper) to map domain errors to responses explicitly, because the default rendering leaks little but is coarse.

**JDK requirement (Blaze).** Running the Blaze backend requires JDK 8u252 or newer (it relies on ALPN APIs unavailable before that); any JDK 9+ works[^1]. Ember does not carry this constraint.

## When to Use / When Not

**Use when:**
- Your stack is already Typelevel (cats-effect, fs2, circe, doobie/skunk) and you want an HTTP layer that composes with it as values.
- You want streaming request/response bodies with principled resource safety.
- You want to target Scala.js or Scala Native for HTTP (Ember), not just the JVM.
- You prefer a small, composable library you assemble yourself over an opinionated framework.

**Avoid when:**
- Your team doesn't know (or doesn't want to learn) cats-effect and tagless-final — the conceptual on-ramp is steep and unavoidable.
- You want a full web framework with templating, ORM, and DI included — that is out of scope by design.
- Your codebase is ZIO-first; `zio-http` fits that runtime without cats-effect interop friction.
- You need a stable, frozen 1.0 API contract today — the recommended series is still pre-1.0.

## Alternatives

- zio/zio-http — use instead when your effect system is ZIO rather than cats-effect; native to that runtime.
- softwaremill/tapir — use to describe endpoints once and derive server (including http4s), client, and OpenAPI; complements more than replaces.
- apache/pekko — use when you want the Akka HTTP model (actors/streams) without an effect-typeclass stack; Akka HTTP's Apache-licensed successor.
- playframework/playframework — use when you want a full MVC framework with routing, templating, and asset pipeline included.
- com-lihaoyi/cask — use for small services/scripts where you want minimal ceremony and no functional-effect prerequisites.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2013-03 | First releases; scalaz-stream bodies, blaze backend. |
| 0.15 | 2016-11 | Last scalaz-based line before the cats migration. |
| 0.16–0.18 | 2017–2018 | Move to cats + fs2; `HttpService` era. |
| 0.20 | 2019-04 | `HttpRoutes`/`HttpApp` model; cats-effect 1/2. |
| 0.21 | 2020-02 | cats-effect 2, fs2 2; long-lived stable line. |
| 0.22 | 2021 | Ember maturing; final cats-effect 2 series. |
| 0.23 | 2021 | cats-effect 3, fs2 3 — current recommended stable series[^2]. |
| 1.0.0-Mx | ongoing | Milestone releases; not yet a final 1.0 (M47 latest)[^2]. |

## References

[^1]: http4s README — project description, partial-unification note, and JDK 8u252 requirement for the Blaze backend. https://github.com/http4s/http4s/blob/series/0.23/README.md
[^2]: http4s release tags — 0.23.x is the current stable series; 1.0.0 remains at milestone tags (M47 latest observed). https://github.com/http4s/http4s/tags

## Tags

scala, http-server, http-client, functional-programming, cats-effect, fs2, typelevel, streaming, jvm, library
