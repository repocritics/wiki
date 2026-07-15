# square/moshi

> A modern JSON library for Android, Java, and Kotlin — Gson's streaming model, minus its footguns, built on Okio.

[GitHub repo](https://github.com/square/moshi) ·
[Official website](https://square.github.io/moshi/1.x/) ·
[License: Apache-2.0](https://github.com/square/moshi/blob/master/LICENSE.txt)

## Overview

Moshi is Square's JSON serialization library for the JVM and Android, first released as 1.0 in 2015[^1]. It shares lineage with Google's Gson — Jesse Wilson, Moshi's lead author, also worked on Gson, OkHttp, and Okio — and deliberately reuses Gson's streaming reader/writer and reflective binding model[^2]. The pitch is not "faster than Jackson" but "the boring parts of JSON, done predictably": a small API surface, explicit type adapters, and no surprising global configuration.

Moshi's defining stance is that it does *less* than its competitors on purpose. It refuses to serialize platform types (`java.*`, `javax.*`, `android.*`) without a user-supplied adapter, so you can't accidentally couple your wire format to a JDK internal. It has no field-naming policy, no versioning system, no `JsonElement` tree model (you get plain `Map`/`List` instead), and no HTML-safe escaping. Every one of those omissions is a Gson feature Moshi chose not to reimplement[^2].

The central tension is Kotlin support. Moshi's reflective core predates Kotlin's rise, and reflecting over Kotlin classes correctly (nullability, default values, `data class` constructors) requires either the `moshi-kotlin` reflection module (which pulls in the ~2 MB `kotlin-reflect`) or the `moshi-kotlin-codegen` annotation processor. Choosing between them — and getting the build wiring right — is the most common source of Moshi friction.

## Getting Started

Gradle (KSP-based code gen, the recommended Kotlin path):

```kotlin
plugins { id("com.google.devtools.ksp") }

dependencies {
  implementation("com.squareup.moshi:moshi:1.15.2")
  ksp("com.squareup.moshi:moshi-kotlin-codegen:1.15.2")
}
```

```kotlin
import com.squareup.moshi.JsonClass
import com.squareup.moshi.Moshi

@JsonClass(generateAdapter = true)
data class Card(val rank: Char, val suit: Suit)

enum class Suit { CLUBS, DIAMONDS, HEARTS, SPADES }

val moshi = Moshi.Builder().build()
val adapter = moshi.adapter(Card::class.java)

val card = adapter.fromJson("""{"rank":"A","suit":"HEARTS"}""")
val json = adapter.toJson(Card('4', Suit.CLUBS))
```

The `@JsonClass(generateAdapter = true)` annotation makes KSP emit a `CardJsonAdapter` at compile time, avoiding reflection entirely. For plain Java classes, `Moshi.Builder().build()` alone is enough — reflection over Java fields needs no extra module.

## Architecture / How It Works

Moshi is built around two interfaces: `JsonReader`/`JsonWriter` (streaming, Okio-backed) and `JsonAdapter<T>` (binding). A `Moshi` instance is an ordered chain of `JsonAdapter.Factory` objects; `adapter(Type)` walks the chain until a factory claims the type, then caches the result. This is the entire extension model — everything from built-in primitives to your custom adapters is a factory.

Custom adapters come in two flavors. The low-level form implements `JsonAdapter<T>` and drives the reader/writer directly. The high-level form is a plain class with `@ToJson`/`@FromJson` methods, which Moshi reflectively adapts; these can chain (a `@FromJson` that takes an intermediate DTO lets Moshi parse to the DTO first, then transform). `@Json(name = ...)` renames a field, `@JsonQualifier` lets one type have context-specific encodings (e.g. an `int` that is a hex color in one field and a plain number elsewhere).

Kotlin support is layered on top rather than baked in:

- **`moshi-kotlin` (reflection)** — `KotlinJsonAdapterFactory` reads Kotlin metadata at runtime via `kotlin-reflect`. Honors nullability and constructor defaults. Simplest to set up, heaviest at runtime, and pulls a large transitive dependency.
- **`moshi-kotlin-codegen` (code gen)** — an annotation processor (kapt originally, KSP since 1.13[^3]) that generates a `*JsonAdapter` per `@JsonClass`. No runtime reflection, R8/ProGuard-friendly, but adds a build step and only sees classes you annotate.

The `moshi-adapters` module ships opt-in helpers that Moshi refuses to include by default: `Rfc3339DateJsonAdapter` for dates, `PolymorphicJsonAdapterFactory` for tagged sealed hierarchies, and `EnumJsonAdapter` with fallback support. Retrofit integration is a separate artifact, `converter-moshi`[^4].

## Production Notes

- **Pick one Kotlin strategy and pin the version.** `moshi`, `moshi-kotlin`, and `moshi-kotlin-codegen` must share a version. Mixing the reflection factory and code gen works but defeats the point; if code gen is present, don't also register `KotlinJsonAdapterFactory` or you pay for reflection you aren't using.
- **`kotlin-reflect` bloats Android APKs.** The reflection module drags in ~2 MB. On Android, prefer code gen; it also plays correctly with R8 shrinking, whereas reflection needs keep rules.
- **Default values are a JVM-semantics trap.** For plain Java classes without a no-arg constructor, a missing JSON field yields `0`/`false`/`null` — *not* the field initializer — because Moshi can't invoke the initializer[^5]. Kotlin code gen and the reflection factory handle constructor defaults correctly; plain Java reflection does not.
- **No date/BigDecimal/etc. out of the box.** Moshi intentionally omits adapters for `java.util.Date`, `java.time`, and platform types. You must register your own or add `moshi-adapters`. This surprises Gson migrants who expect dates to "just work."
- **Unknown fields are skipped silently by default.** Use `failOnUnknown()` if you want strict decoding. Conversely, malformed input throws `IOException`; well-formed-but-wrong-shape input throws `JsonDataException` with a JSON path (`$.visible_cards[2].suit`), which is excellent for debugging.
- **kapt is slow; migrate to KSP.** Teams on the older kapt code gen see meaningful build-time wins moving to KSP (1.13+). The generated adapter output is equivalent.
- **`@JvmSuppressWildcards` / generics.** Reified `moshi.adapter<List<Card>>()` in Kotlin captures the full type; the Java equivalent needs `Types.newParameterizedType(...)`. Getting generic collection types wrong is a frequent runtime `IllegalArgumentException`.

## When to Use / When Not

**Use when:**
- You're on Android or Kotlin and want compile-time, reflection-free JSON via KSP.
- You want a small, predictable API and are willing to write explicit adapters for edge types.
- You already use OkHttp/Okio/Retrofit — Moshi shares Okio buffers with OkHttp for efficiency.
- You want decoding errors that pinpoint the offending JSON path.

**Avoid when:**
- You need maximum throughput on very large payloads or a vast module ecosystem — Jackson is the heavier-duty choice.
- You want zero-config parsing of arbitrary types including dates and platform classes — Gson is more permissive.
- You're building Kotlin Multiplatform — Moshi is JVM/Android only; use kotlinx.serialization.
- You rely on a mutable JSON tree model (`JsonElement`-style) for ad-hoc manipulation — Moshi gives you `Map`/`List` instead.

## Alternatives

- google/gson — use when you want lenient, zero-config parsing and don't mind runtime reflection; Moshi's direct ancestor and easiest migration target.
- FasterXML/jackson — use when you need top throughput, streaming for huge documents, and a large module ecosystem (`jackson-datatype-jsr310`, etc.).
- Kotlin/kotlinx.serialization — use when you're Kotlin Multiplatform or prefer a compiler plugin over KSP/reflection.
- square/wire — use when your wire format is Protobuf rather than JSON (also from Square, same design philosophy).
- square/okio — not an alternative but Moshi's I/O foundation; worth knowing if you tune buffers.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2014-08 | Repository created[^6]. |
| 1.0 | 2015 | First stable release. Okio-backed streaming, Gson-derived binding[^1]. |
| 1.8 | 2019 | Kotlin support hardening; `@Json(ignore = true)` and related refinements. |
| 1.9 | 2019 | AutoValue extension split out; continued Kotlin code-gen maturation. |
| 1.13 | 2022 | KSP support added alongside kapt for `moshi-kotlin-codegen`[^3]. |
| 1.15.x | 2023–2024 | Current 1.x line; KSP-first Kotlin code gen. |

## References

[^1]: Moshi README and project site, "A modern JSON library for Android, Java and Kotlin." https://square.github.io/moshi/1.x/
[^2]: Moshi README, "Borrows from Gson" — differences from Gson (fewer adapters, no naming policy, no `JsonElement`, no HTML-safe escaping). https://github.com/square/moshi#borrows-from-gson
[^3]: Moshi code gen documentation on KSP/kapt annotation processing. https://github.com/square/moshi#codegen
[^4]: Retrofit Moshi converter, `com.squareup.retrofit2:converter-moshi`. https://github.com/square/retrofit/tree/master/retrofit-converters/moshi
[^5]: Moshi README, "Default Values & Constructors" — missing-field behavior for Java classes without a no-arg constructor. https://github.com/square/moshi#default-values--constructors
[^6]: GitHub API, repos/square/moshi metadata (created 2014-08-09, license Apache-2.0). https://github.com/square/moshi

## Tags

json, kotlin, java, android, serialization, ksp, okio, jvm, code-generation, library
