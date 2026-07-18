# circe/circe

> The default JSON library of functional Scala — cats-based codecs, a safe
> immutable AST, and generic derivation, still shipping as 0.14.x a decade in.

[GitHub repo](https://github.com/circe/circe) ·
[Official guide](https://circe.github.io/circe/) ·
[License: Apache-2.0](https://github.com/circe/circe/blob/series/0.14.x/LICENSE)

## Overview

circe is a JSON library for Scala and Scala.js, forked from Argonaut in 2015 by
Travis Brown and rebuilt on the cats/Typelevel stack[^1]. It provides an
immutable `Json` AST, `Encoder[A]` / `Decoder[A]` type classes, a zipper-style
cursor API for traversal, and compile-time generic derivation of codecs for
case classes and sealed trait hierarchies. In the cats/http4s/fs2 ecosystem it
is the assumed JSON layer: http4s ships `http4s-circe`, and a long tail of
libraries (sttp, Github4s, elastic4s, Snowplow, Scio) integrate with it
directly[^2]. Adopters listed in the README include Stripe, Spotify, Twilio,
SoundCloud, and The Guardian[^2].

The defining tradeoff is principled totality over raw speed. Every operation
returns `Either` or `Validated` — no exceptions, no nulls, no reflection — and
decoding failures carry a cursor history that pinpoints the failing path.
The cost is an intermediate AST on every parse/print and type-class dispatch
on every field, which makes circe measurably slower than macro-generated
direct codecs (jsoniter-scala) on hot paths.

The project's second tension is versioning. The default branch is literally
`series/0.14.x`; the 0.14 line has been current since mid-2021[^3], and a 1.0
release is referenced in the README only in the future tense. Maintenance is
real — pushed to within days as of mid-2026, currently led by Darren Gibson
and Erlend Hamnaberg after Brown's departure[^2] — but feature development is
conservative; a decade of production use has not produced a stable major
version, and binary compatibility is instead managed within the 0.14 series.
2.5k stars understates its reach: it is infrastructure, installed via build
files, not starred.

## Getting Started

```scala
// build.sbt
val circeVersion = "0.14.10" // any recent 0.14.x
libraryDependencies ++= Seq(
  "io.circe" %% "circe-core",
  "io.circe" %% "circe-generic",
  "io.circe" %% "circe-parser"
).map(_ % circeVersion)
```

```scala
import io.circe._, io.circe.generic.semiauto._
import io.circe.parser.decode, io.circe.syntax._

case class User(id: Long, name: String)
implicit val userCodec: Codec[User] = deriveCodec[User]  // Scala 2 semiauto

val out: String = User(1, "Ada").asJson.noSpaces
// {"id":1,"name":"Ada"}
val back: Either[Error, User] = decode[User](out)
```

On Scala 3, derivation uses `Mirror`s instead of shapeless:
`case class User(id: Long, name: String) derives Codec.AsObject`.

## Architecture / How It Works

circe is a small constellation of modules with strict layering:

- **circe-core** — the `Json` AST (null, boolean, number, string, array,
  object), `Encoder`/`Decoder`/`Codec` type classes, and the cursor API.
  `JsonNumber` is its own abstraction: parsed numbers are kept in a lazy
  precision-preserving representation rather than eagerly converted to
  `Double` or `BigDecimal`, which both avoids silent precision loss and blunts
  pathological-exponent inputs.
- **circe-parser** — parsing, delegated to Jawn on the JVM and a hand-written
  parser on Scala.js. circe does not own a tokenizer on the JVM.
- **circe-generic** — derivation. Scala 2 goes through shapeless
  (`LabelledGeneric`); Scala 3 uses compiler `Mirror`s via `io.circe.derivation`.
  Two modes: `auto` (implicit derivation at every use site) and `semiauto`
  (explicit `deriveEncoder`/`deriveDecoder` cached in a `val`).
- **circe-literal** — a `json"..."` string interpolator validated at compile
  time.

Traversal is a zipper: `hcursor.downField("a").downN(2).as[Int]` navigates and
records every step. On failure, the `DecodingFailure` carries the accumulated
`CursorOp` history, so errors read as paths ("attempt to decode Int at
.a[2]") rather than opaque messages. `Decoder` forms a monad with error
handling, so custom decoders compose with `flatMap`/`or`/`emap`; an
alternative `decodeAccumulating` path returns `ValidatedNel` to report all
failures instead of the first.

The coupling story: circe-core depends on cats-core, so its major-version
cadence is chained to cats (which has been similarly stable). Optics
(`circe-optics`, Monocle-based) and configurable derivation for Scala 2
(`circe-generic-extras`) live in separate repos on their own release trains —
a recurring source of version-skew confusion after 0.14 splits.

## Production Notes

**Auto derivation is a compile-time trap.** `import io.circe.generic.auto._`
re-derives codecs implicitly at every call site; on Scala 2 the shapeless
machinery can add minutes to compile times and produce megabytes of bytecode
in large codebases. The near-universal production convention is semiauto with
codecs pinned in companion objects. Treat `auto` as demo-ware.

**Throughput.** circe parses to a full AST and then decodes from it; printing
is the mirror image. Community JVM benchmarks (circe-benchmarks,
jsoniter-scala-benchmark) consistently place circe well behind
jsoniter-scala's direct-to-case-class macro codecs, and roughly in Jackson's
neighborhood depending on shape[^4]. It is fast enough for most services —
Criteo reported ~200k events/s after migrating from Jackson[^2] — but for
serialization-bound hot paths, jsoniter-scala-circe exists precisely to swap
the parser/printer under circe's API.

**Scala 2 vs Scala 3 derivation are different machines.** Shapeless-based
`generic-extras` configuration (snake_case keys, defaults, discriminators)
does not carry to Scala 3; the 0.14.x series grew a parallel
`Configuration`-based derivation (`ConfiguredCodec`) for Scala 3 instead.
Cross-building codebases must be tested on both — derived output can differ
in edge cases (e.g. handling of defaults and empty objects).

**Pre-1.0 forever (so far).** Within `series/0.14.x`, binary compatibility is
enforced (MiMa) and upgrades are boring. Across series (0.12 → 0.13 → 0.14),
expect source breakage and, historically, lockstep upgrades with cats and the
rest of the Typelevel stack. Mixed-series classpaths fail at runtime with
`NoSuchMethodError`; align every `io.circe` artifact plus transitive users
(http4s, sttp) on one series.

**Numbers and keys.** Decoding into `Double` silently loses precision that
the AST itself preserved — use `BigDecimal`/`JsonNumber` when it matters.
JSON object key order is preserved by `JsonObject`, but relying on it is a
smell other libraries won't honor.

## When to Use / When Not

**Use when:**
- You are already in the cats/http4s/fs2/Typelevel ecosystem — it is the
  path of least resistance and best-integrated option.
- You want total, exception-free decoding with precise error paths and
  optional error accumulation for API validation.
- You need Scala.js support with the same codecs on both sides.
- Sealed-trait ADT hierarchies dominate your wire format.

**Avoid when:**
- Serialization is your bottleneck: jsoniter-scala is several times faster
  and allocation-free by design.
- You want minimal dependencies or fast compiles in a non-cats codebase —
  upickle is lighter on both axes.
- You interop heavily with Java frameworks expecting Jackson modules.
- You need a stable 1.x semver contract; circe's contract is "the current
  0.14 series", which some organizations' policies reject.

## Alternatives

- plokhotnyuk/jsoniter-scala — use instead when throughput/allocation matters;
  macro codecs go straight to case classes with no AST.
- com-lihaoyi/upickle — use instead for small dependency footprint, fast
  compiles, and the lihaoyi (os-lib/cask/mill) ecosystem.
- zio/zio-json — use instead in ZIO codebases; security-conscious design,
  no cats dependency.
- argonaut-io/argonaut — circe's ancestor; only for legacy scalaz codebases.
- FasterXML/jackson-module-scala — use instead when Java-framework interop
  (Spring, Kafka tooling) dictates Jackson.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2015-09 | First release, as a fork of Argonaut on cats (briefly named "jfc" before the rename)[^1]. |
| 0.9.0 | 2018-01 | cats 1.0 alignment. |
| 0.12.0 | 2019-09 | cats 2.0, Scala 2.13 era. |
| 0.14.0 | 2021-05 | Scala 3 support via Mirror-based derivation; start of the long-running 0.14 series[^3]. |
| 0.14.x | 2021–2026 | Maintenance series: Scala 3 `Configuration` derivation, Jawn/cats bumps, MiMa-enforced binary compat. Still pre-1.0. |

## References

[^1]: circe guide — project background and Argonaut lineage. https://circe.github.io/circe/
[^2]: circe README — adopters, maintainers, ecosystem projects. https://github.com/circe/circe#readme
[^3]: circe v0.14.0 release. https://github.com/circe/circe/releases/tag/v0.14.0
[^4]: jsoniter-scala benchmark suite (includes circe). https://github.com/plokhotnyuk/jsoniter-scala

## Tags

scala, json, serialization, codec, functional-programming, cats, typelevel, generic-derivation, scala-js, parsing
