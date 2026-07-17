# FasterXML/jackson-modules-java8

> Jackson 2.x add-on modules that teach ObjectMapper to serialize Java 8 datatypes — `Optional`, `java.time`, and constructor parameter names.

[GitHub repo](https://github.com/FasterXML/jackson-modules-java8) ·
License: Apache-2.0

## Overview

This repository is a multi-module umbrella project that supplies the Java 8 support Jackson 2.x cannot ship in its core because that core still targets Java 7 (and, before 2.7, Java 6)[^1]. It packages three independent Jackson modules, each a separate Maven artifact: `jackson-datatype-jsr310` (JSR-310 `java.time` types), `jackson-datatype-jdk8` (`Optional`, `OptionalLong`, `OptionalDouble`, and other non-temporal Java 8 additions), and `jackson-module-parameter-names` (reads real constructor/factory parameter names from bytecode so you can deserialize immutable types without annotating every field with `@JsonProperty`).

The defining tension is that these are compatibility shims for a language-version gap, not permanent features. Jackson 3.0 requires Java 17, so it folds all three modules directly into `jackson-databind` — they no longer need to exist or be registered as of 3.0.0[^2]. That makes this repo one of the clearest examples in the Jackson ecosystem of a subproject with a scheduled end of life: it is maintained for the 2.x line only. `jackson-datatype-jsr310` in particular is one of the most-depended-upon artifacts in the entire JVM ecosystem, because almost any 2.x service that touches `LocalDate`/`Instant` over JSON pulls it in.

The most common reason people arrive here is not a feature request but a serialization surprise: without the date/time module registered, Jackson writes `java.time` values as opaque object graphs or numeric timestamps rather than ISO-8601 strings. Understanding which module you need, and the difference between the two date/time module variants, resolves the majority of those issues.

## Getting Started

With Maven (versions are best pinned via the Jackson BOM rather than declared individually):

```xml
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jsr310</artifactId>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.datatype</groupId>
    <artifactId>jackson-datatype-jdk8</artifactId>
</dependency>
<dependency>
    <groupId>com.fasterxml.jackson.module</groupId>
    <artifactId>jackson-module-parameter-names</artifactId>
</dependency>
```

```java
ObjectMapper mapper = JsonMapper.builder()
    .addModule(new JavaTimeModule())       // NOT the legacy JSR310Module
    .addModule(new Jdk8Module())
    .addModule(new ParameterNamesModule())
    // write java.time as ISO-8601 strings, not numeric arrays
    .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
    .build();

record Event(String name, Optional<String> note, Instant at) {}

String json = mapper.writeValueAsString(
    new Event("launch", Optional.empty(), Instant.parse("2026-07-17T00:00:00Z")));
// {"name":"launch","note":null,"at":"2026-07-17T00:00:00Z"}
```

Or let Jackson discover the modules on the classpath via `ServiceLoader`:

```java
ObjectMapper mapper = new ObjectMapper();
mapper.findAndRegisterModules();
```

## Architecture / How It Works

Each module is a `com.fasterxml.jackson.databind.Module` that registers serializers and deserializers for a set of types. They are otherwise unrelated codebases sharing a build; the parent `jackson-modules-java8` POM is a build aggregator only and deliberately has no dependency on its children, so depending on the parent artifact gets you nothing usable[^1].

The date/time module ships in two variants that differ only in default configuration. `JavaTimeModule` is the current, recommended one; `JSR310TimeModule` (the older `JSR310Module`) exists for backwards compatibility with code that predates the config changes. Both cover the same JSR-310 types. The trap is auto-registration history: before Jackson 2.10, `findAndRegisterModules()` picked up the legacy `JSR310Module`, not `JavaTimeModule`. From 2.10 onward auto-discovery registers `JavaTimeModule`[^1]. If you need the non-default variant while still auto-registering, register it explicitly after the discovery call — the differing `getTypeId()` values keep it from being treated as a duplicate.

Registration ordering has real semantics. Do not mix explicit and automatic registration: if you do, only one wins, and which one depends on settings. With `MapperFeature.IGNORE_DUPLICATE_MODULE_REGISTRATIONS` set, the first registration wins and later ones are dropped (duplicates identified by `Module.getTypeId()`); without it, the last registration wins because later registrations override earlier ones[^1].

`jackson-module-parameter-names` is the one with a hard runtime prerequisite: it reads parameter names emitted into bytecode, which only exist if the code was compiled with `javac -parameters`. Without that flag the module silently provides nothing and creator detection falls back to annotations.

## Production Notes

**`java.time` defaults will bite you.** With `WRITE_DATES_AS_TIMESTAMPS` at its default-enabled state, `Instant` and friends serialize as numeric arrays / epoch values rather than ISO-8601 strings. Nearly every real service disables that feature. This is the single most frequent support issue for the module.

**Precision and timezone handling are configurable and surprising.** Nanosecond precision (`WRITE_DATE_TIMESTAMPS_AS_NANOSECONDS`), whether `ZonedDateTime` retains its zone id, and context timezone adjustment (`ADJUST_DATES_TO_CONTEXT_TIME_ZONE`) all have non-obvious defaults. Round-tripping a `ZonedDateTime` and getting back a different zone offset is a known category of bug caused by leaving these at defaults.

**Version alignment is mandatory.** These modules must match your `jackson-databind` version exactly. Mixing, say, a 2.15 databind with a 2.17 datetime module produces `NoClassDefFoundError`/`AbstractMethodError` at runtime rather than a build failure. Always import `jackson-bom` and let it govern the version set instead of pinning artifacts by hand.

**The parameter-names footgun is a build-config gap, not a code bug.** If constructor-based deserialization of records or immutable value types "randomly" fails only in some build environments, check whether `-parameters` is set consistently across local, CI, and any annotation-processing paths. Spring Boot enables it by default via its Maven/Gradle plugins, which masks the requirement until you build outside that toolchain.

**This is 2.x-only, and that is the biggest upgrade note.** When you move to Jackson 3.0 you must remove all three dependencies and delete the `registerModule`/`addModule` calls — the functionality is built into `jackson-databind` 3.0 and re-registering the old modules is at best redundant. Plan the 3.x migration as a code change, not just a version bump. The Java baseline also jumps to 17 at that boundary.

## When to Use / When Not

**Use when:**
- You run Jackson 2.x and serialize `java.time` types — effectively non-optional for correct ISO-8601 output.
- You deserialize records or immutable constructors and want to avoid `@JsonProperty` on every parameter (add `parameter-names` plus `-parameters`).
- Your API contract exposes `Optional<T>` fields and you want `null`/absent handled coherently (`jdk8` module).

**Avoid / skip when:**
- You are on Jackson 3.x — these are already inside `jackson-databind`; adding them is wrong.
- You only handle primitives and `String`s over JSON and never touch `java.time` or `Optional` — core Jackson already covers you.
- You need the modules but forgot the version discipline — an unaligned add is worse than not adding, because it fails at runtime.

## Alternatives

- FasterXML/jackson-databind — the core; on Jackson 3.x it already contains this functionality, so "use it instead" is the correct path post-3.0.
- google/gson — use instead when you want a smaller, annotation-light JSON library and can register a `java.time` `TypeAdapter` yourself.
- eclipse-ee4j/jsonb-api (JSON-B) — use instead when you want the Jakarta EE standard binding API with `java.time` support built into the spec rather than a bolt-on module.
- FasterXML/jackson-datatype-joda — use instead when your domain model still uses Joda-Time types rather than JSR-310.
- ngs-doo/dsl-json — use instead when serialization throughput matters more than the Jackson ecosystem and you can accept compile-time code generation.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo split out | 2016-11 | Java 8 modules consolidated into this umbrella repo[^3]. |
| 2.10 | 2019-09 | Auto-registration switches to `JavaTimeModule` over legacy `JSR310Module`[^1]. |
| 2.x line | ongoing | All three modules maintained here for the Java 7-baseline 2.x releases[^1]. |
| 3.0.0 | 2025 | Modules merged into `jackson-databind`; Java 17 required; repo no longer needed for 3.x[^2]. |

## References

[^1]: Project README, FasterXML/jackson-modules-java8 — module list, registration rules, and the 2.10 auto-registration change. https://github.com/FasterXML/jackson-modules-java8
[^2]: Project README — Jackson 3.x section: modules merged into `jackson-databind`, Java 17 requirement. https://github.com/FasterXML/jackson-modules-java8#jackson-3x
[^3]: GitHub repository metadata, FasterXML/jackson-modules-java8 (created 2016-11-03). https://github.com/FasterXML/jackson-modules-java8

## Tags

java, jackson, json, serialization, java-time, jsr-310, optional, data-binding, jvm, apache-2.0
