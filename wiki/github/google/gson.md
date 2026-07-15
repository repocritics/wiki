# google/gson

> Reflection-based Java library that converts arbitrary Java objects to JSON and back — now in maintenance mode.

[GitHub repo](https://github.com/google/gson) ·
[License: Apache-2.0](https://github.com/google/gson/blob/main/LICENSE)

## Overview

Gson is a Java serialization/deserialization library that maps Java objects to JSON and back[^1]. Its founding design goal, dating to Google's 2008 open-source release, was to serialize objects you *don't own* — classes with no annotations, no default constructor, and deep generic hierarchies — without touching their source[^2]. That set it apart from annotation-driven libraries of the era and made it the default JSON tool in a generation of Android and server-side Java codebases.

The defining tradeoff is **reflection**. Gson reads and writes fields by reflecting over class structure at runtime, with no annotations or code generation required. This is what makes it ergonomic (`new Gson().toJson(obj)` just works) and also what makes it fragile: any tool that renames, removes, or strips fields — R8, ProGuard, obfuscators — silently breaks Gson at runtime rather than at compile time. The maintainers now explicitly recommend *against* Gson for Android release builds and for non-Java JVM languages[^3].

As of 2026 Gson is in **maintenance mode**: existing bugs are fixed, but large new features are not planned[^3]. It remains extremely widely deployed (transitively pulled in by a large fraction of the JVM ecosystem), which is precisely why its stability guarantees and its footguns both matter.

## Getting Started

Gradle:

```gradle
dependencies {
  implementation 'com.google.code.gson:gson:2.14.0'
}
```

Maven:

```xml
<dependency>
  <groupId>com.google.code.gson</groupId>
  <artifactId>gson</artifactId>
  <version>2.14.0</version>
</dependency>
```

```java
import com.google.gson.Gson;
import com.google.gson.reflect.TypeToken;
import java.lang.reflect.Type;
import java.util.List;

record User(int id, String name) {}

Gson gson = new Gson();

// object -> JSON
String json = gson.toJson(new User(1, "Tom"));   // {"id":1,"name":"Tom"}

// JSON -> object
User back = gson.fromJson(json, User.class);

// generics survive erasure via the super-type-token trick
Type listType = new TypeToken<List<User>>(){}.getType();
List<User> users = gson.fromJson("[{\"id\":1,\"name\":\"Tom\"}]", listType);
```

## Architecture / How It Works

Gson has three layers that sit on top of one streaming core:

1. **Streaming** — `JsonReader` / `JsonWriter` are pull-based, incremental parsers/emitters. Everything else is built on them, and you can drop to this layer for large documents that shouldn't be materialized.
2. **Tree** — `JsonElement` and its subtypes (`JsonObject`, `JsonArray`, `JsonPrimitive`, `JsonNull`) form an in-memory DOM. `JsonParser` builds it; useful for ad-hoc traversal.
3. **Binding** — the object-mapping layer most people use. A `Gson` instance holds a registry of `TypeAdapter<T>` instances produced by `TypeAdapterFactory` implementations.

The binding layer's workhorse is `ReflectiveTypeAdapterFactory`: for a type with no registered adapter, it reflects over declared fields, builds an adapter, and caches it. Generic types are captured with `TypeToken<T>` — an anonymous subclass whose superclass type argument is read reflectively (the "super type token" pattern), the standard workaround for Java's type erasure.

Two coupling points are worth knowing. First, to instantiate classes lacking a no-arg constructor, Gson reaches for `sun.misc.Unsafe` (via `jdk.unsafed`); this can be disabled with `GsonBuilder.disableJdkUnsafe()` but is on by default. Second, `Gson` instances are immutable and thread-safe once built via `GsonBuilder`, so the intended pattern is to construct one and share it — but adapter resolution happens by reflecting live class metadata, which is exactly what shrinkers rewrite.

## Production Notes

**Android / R8 / ProGuard is the central footgun.** Gson's runtime reflection assumes field names and members survive to runtime. Release-mode shrinking renames or removes them, producing empty objects, `null` fields, or crashes that never appear in debug builds. Fixes require explicit `-keep` rules or `@Keep` annotations on every serialized model. The maintainers now steer Android users to Moshi codegen or kotlinx.serialization instead[^3].

**HTML escaping is on by default.** Gson escapes `<`, `>`, `&`, `=`, and `'` to Unicode sequences so output is safe to embed in HTML. This surprises people producing non-web JSON (values look mangled); disable with `GsonBuilder.disableHtmlEscaping()`.

**Null and default semantics.** Fields that are `null` are omitted from output unless you call `serializeNulls()`. On deserialization, missing JSON keys leave fields at their default (Java `null` / `0`), so a "successful" parse can quietly yield a half-populated object with no error.

**Dates and locales.** The default date adapter formats using the default locale and timezone, so serialized output is environment-dependent and round-trips can drift across machines. Pin a format/timezone explicitly, or register a custom adapter (Java 8 `java.time` types have no built-in adapter and need registration).

**Leniency has shifted between versions.** Gson historically parsed some malformed JSON in "lenient" mode; leniency defaults were tightened in newer releases, so upgrades can turn previously-accepted input into parse errors. Check `setStrictness` / `JsonReader` behavior when upgrading.

**Security.** CVE-2022-25647 (deserialization gadget via internal `writeReplace`) was fixed in 2.8.9; stay at 2.8.9 or later[^4]. As with any reflective deserializer, never bind untrusted JSON directly into arbitrary polymorphic types.

**Performance.** Reflection-per-type is resolved once and cached, but first-use adapter construction and the absence of codegen make Gson slower than Jackson or codegen-based Moshi in throughput-sensitive paths. For most CRUD/config workloads this is irrelevant; for hot serialization loops it is measurable.

## When to Use / When Not

**Use when:**
- You're on server-side Java and want zero-ceremony JSON for objects you may not own.
- You need broad Java generics support without annotating model classes.
- You're maintaining an existing codebase already standardized on Gson (its stability in maintenance mode is a feature).

**Avoid when:**
- You ship Android release apps with R8/ProGuard — reflection and shrinking fight each other.
- Your models are Kotlin or Scala — language features like non-null types and default arguments are unsupported and fail confusingly[^3].
- You need maximum throughput or active feature development — Gson is frozen; alternatives iterate.

## Alternatives

- FasterXML/jackson-databind — the enterprise default; faster, far more features and modules, but heavier and more annotation-driven.
- square/moshi — modern, Android-friendly, offers reflection *or* compile-time codegen; APIs feel familiar to Gson users.
- Kotlin/kotlinx.serialization — compile-time, multiplatform, the right choice for Kotlin projects.
- alibaba/fastjson2 — high performance on the JVM, though the fastjson lineage has a heavier CVE history to weigh.
- eclipse-vertx/vert.x or built-in `jakarta.json` (JSON-P/JSON-B) — standards-based option when you want a spec, not a library.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2008 | First open-source release by Google (moved from Google Code)[^2]. |
| 2.0 | 2011 | Major API revision; streaming reader/writer core. |
| 2.8.0 | 2017 | Long-lived line; `TypeToken` / adapter API stabilized. |
| 2.8.9 | 2021 | Fixes CVE-2022-25647; Java 6 min[^4]. |
| 2.10.1 | 2023 | Bug-fix release on the 2.10 line. |
| 2.11.0 | 2024 | Android minSdk raised to API 21. |
| 2.12.0 | 2025 | Minimum Java version raised to 8. |
| 2.14.0 | 2026 | Current release on Maven Central[^1]. |

## References

[^1]: google/gson README and Maven Central artifact. https://github.com/google/gson · https://central.sonatype.com/artifact/com.google.code.gson/gson
[^2]: Gson design document (history and goals). https://github.com/google/gson/blob/main/GsonDesignDocument.md
[^3]: google/gson README — maintenance-mode notice and Android/Kotlin guidance. https://github.com/google/gson#gson
[^4]: CVE-2022-25647, deserialization of untrusted data in Gson before 2.8.9. https://nvd.nist.gov/vuln/detail/CVE-2022-25647

## Tags

java, json, serialization, deserialization, reflection, jvm, android, data-binding, maintenance-mode, google
