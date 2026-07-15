# Kotlin/kotlinx.serialization

> Compile-time, reflectionless serialization for Kotlin — one `@Serializable` annotation, many formats, all Kotlin targets.

[GitHub repo](https://github.com/Kotlin/kotlinx.serialization) ·
[Serialization Guide](https://kotlinlang.org/api/kotlinx.serialization/) ·
[License: Apache-2.0](https://github.com/Kotlin/kotlinx.serialization/blob/master/LICENSE)

## Overview

kotlinx.serialization is JetBrains' first-party serialization framework for Kotlin. It is two pieces that ship separately: a **Kotlin compiler plugin** that generates a serializer for every class you mark `@Serializable`, and a **runtime library** (`kotlinx-serialization-core`) plus per-format modules (JSON, Protobuf, CBOR, HOCON, Properties)[^1]. The defining choice is that serializers are generated at compile time rather than discovered by reflection — the same reason it works uniformly on JVM, JS, Native, and Wasm, where reflection is limited or absent.

The audience is Kotlin developers who want a type-safe, multiplatform-native codec without the reflection and annotation-processing baggage of the Java ecosystem (Jackson, Gson, Moshi). The tradeoff is that its metaprogramming lives inside the Kotlin compiler, so the plugin version is locked in lockstep with the Kotlin compiler version, while the runtime library versions independently[^2]. This split — plugin tied to Kotlin, runtime on its own `1.x` line — is the single most common source of confusion and dependency friction for newcomers.

It is a JetBrains "official" and Stable-tier project[^1], meaning API stability guarantees apply to the core, though individual format modules (Protobuf, CBOR) and features (`SerializersModule`, sealed-class polymorphism) matured at different rates.

## Getting Started

Two things are required: the plugin (matched to your Kotlin version) and a format dependency (versioned separately).

```kotlin
// build.gradle.kts
plugins {
    kotlin("jvm") version "2.3.20"
    kotlin("plugin.serialization") version "2.3.20" // must match the Kotlin version
}
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.11.0")
}
```

```kotlin
import kotlinx.serialization.*
import kotlinx.serialization.json.*

@Serializable
data class Project(val name: String, val language: String)

fun main() {
    val json = Json.encodeToString(Project("kotlinx.serialization", "Kotlin"))
    println(json) // {"name":"kotlinx.serialization","language":"Kotlin"}
    val obj = Json.decodeFromString<Project>(json)
    println(obj)  // Project(name=kotlinx.serialization, language=Kotlin)
}
```

## Architecture / How It Works

The compiler plugin runs during Kotlin compilation. For each `@Serializable` class it synthesizes a nested `$serializer` object implementing `KSerializer<T>`, plus a `SerialDescriptor` describing the class's fields, order, and metadata. There is no runtime reflection in the hot path — encoding walks the descriptor and calls `Encoder`/`Decoder` methods that each format implements[^3].

The three-layer model matters:

- **`SerialDescriptor`** — the compile-time schema (field names, kinds, nullability, `@SerialName` overrides). Formats read this to decide wire structure.
- **`Encoder` / `Decoder`** — format-agnostic visitor interfaces. JSON, Protobuf, CBOR each provide an implementation; a custom format is "just" another `Encoder`/`Decoder` pair.
- **`SerializersModule`** — the runtime registry for polymorphism and contextual serializers, since those cannot always be resolved at compile time.

Polymorphism is the sharpest edge. **Sealed** hierarchies get polymorphic serializers automatically, emitting a `"type"` class discriminator. **Open/abstract** hierarchies and interfaces require explicit registration in a `SerializersModule` with `polymorphic { subclass(...) }` — forget it and you get a runtime `SerializationException`, not a compile error. Third-party types you don't own (e.g. `java.time`, `BigDecimal`) need a custom `KSerializer` or `@Contextual` plus a module entry, because the plugin can only generate code for classes it compiles.

`Json` is configured via an immutable options object: `ignoreUnknownKeys`, `isLenient`, `encodeDefaults`, `explicitNulls`, `coerceInputValues`, `classDiscriminator`, `prettyPrint`. Defaults are deliberately strict — unknown keys throw, and default values are omitted from output (`encodeDefaults = false`) — which surprises people migrating from Gson/Jackson's permissive defaults.

## Production Notes

**Version lockstep is the top footgun.** The `plugin.serialization` version must equal your Kotlin compiler version; the runtime `kotlinx-serialization-json` version is independent. Bumping Kotlin without bumping the plugin, or pinning a runtime library newer than the plugin supports, produces cryptic "unsupported version" or missing-serializer errors.

**Android/R8 and named companions.** Bundled ProGuard/R8 rules keep serializers for retained `@Serializable` classes automatically — but they do **not** cover classes with *named* companion objects, whose serializers are looked up via `getDeclaredClasses`. Those need hand-written `-keep` rules, and the rules differ between R8 compatibility mode and full mode[^4]. This bites Android teams late, only after minification strips a serializer that worked in debug builds.

**Strict-by-default parsing.** For evolving external APIs you almost always want `Json { ignoreUnknownKeys = true }`; the default throws on any field not in your model. `coerceInputValues = true` is the escape hatch for null-into-non-null and unknown-enum cases. `explicitNulls = false` is commonly needed to stop null fields from being written.

**Performance.** JSON encoding/decoding is competitive but not the fastest option on the JVM — reflection/codegen libraries like Jackson can edge it out on raw throughput in some benchmarks. The wins are correctness, multiplatform reach, and no reflection warm-up, not peak JVM speed. Very large documents are parsed eagerly by default; streaming (`decodeFromStream`) is JVM-only and comparatively newer.

**Generics and type erasure.** `encodeToString(value)` needs a statically known type. Erased/reflective cases require obtaining a `KSerializer` via `serializer(kType)`, which on JVM does use reflection and is the one place the reflectionless promise bends.

## When to Use / When Not

**Use when:**
- You target Kotlin Multiplatform (JVM + JS + Native + Wasm) and want one codec everywhere.
- You want compile-time-checked serialization with no reflection and no annotation processor.
- You need multiple wire formats (JSON, Protobuf, CBOR) behind one model definition.
- You value strict, predictable parsing and explicit polymorphism.

**Avoid when:**
- You're JVM-only and depend on Jackson's vast module ecosystem (databind, XML, CSV, JSON Schema).
- You need to serialize types you can't annotate and don't want to write custom serializers or contextual modules.
- Your model is highly dynamic/schemaless and you'd rather manipulate a tree than map to classes.
- You can't tolerate the Kotlin-version-lockstep upgrade cadence in a large multi-module build.

## Alternatives

- FasterXML/jackson-module-kotlin — use on JVM-only projects that need the deepest format/module ecosystem and reflection-based flexibility.
- google/gson — use for small, simple JVM JSON needs or legacy Java interop where a heavy toolchain isn't wanted.
- square/moshi — use on Android/JVM when you want reflection-optional codegen adapters and Moshi's smaller footprint.
- charleskorn/kaml — not a competitor but a YAML `Encoder`/`Decoder` built on kotlinx.serialization; use it to add YAML to an existing `@Serializable` model.
- Kotlin/kotlinx-io — pair with it for buffered/streaming I/O under the serialization layer, not a serializer itself.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2017–2019 | Experimental; API and coordinates changed repeatedly, tracked pre-release Kotlin[^5]. |
| 1.0.0 | 2020-10 | First stable release; API guarantees, `SerializersModule`, sealed polymorphism[^6]. |
| 1.x | 2021–2026 | Incremental: value-class support, `explicitNulls`, JSON streaming (JVM), format hardening. |
| 1.11.0 | 2026 | Current runtime line; plugin tracks Kotlin 2.3.20[^1]. |

## References

[^1]: kotlinx.serialization README — "Kotlin multiplatform / multi-format reflectionless serialization", Stable / JetBrains-official badges, current versions. https://github.com/Kotlin/kotlinx.serialization
[^2]: README, "Setup" — plugin released in tandem with each Kotlin compiler version; runtime library has different coordinates and versioning. https://github.com/Kotlin/kotlinx.serialization#setup
[^3]: Kotlin Serialization Guide — serializers, `SerialDescriptor`, `Encoder`/`Decoder`. https://kotlinlang.org/api/kotlinx.serialization/
[^4]: README, "Android" — ProGuard/R8 rules and the named-companion-object caveat. https://github.com/Kotlin/kotlinx.serialization#android
[^5]: Repository created 2017-07-20 (GitHub API); early releases were experimental and pre-1.0.
[^6]: kotlinx.serialization 1.0.0 stable release, October 2020. https://github.com/Kotlin/kotlinx.serialization/releases

## Tags

kotlin, serialization, json, protobuf, cbor, multiplatform, compiler-plugin, jvm, kotlin-native, jetbrains
