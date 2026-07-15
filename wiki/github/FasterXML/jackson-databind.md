# FasterXML/jackson-databind

> The data-binding layer of Jackson — POJO ↔ JSON mapping, the tree model, and the `ObjectMapper` that most of the Java ecosystem serializes through.

[GitHub repo](https://github.com/FasterXML/jackson-databind) ·
[License: Apache-2.0](https://github.com/FasterXML/jackson-databind/blob/3.x/LICENSE)

## Overview

Jackson is Java's dominant JSON toolchain, split across three coordinated repos: `jackson-core` (the streaming parser/generator), `jackson-annotations` (configuration metadata), and this one, `jackson-databind`, which sits on top and provides the `ObjectMapper`, the `JsonNode` tree model, and the reflection-driven serializer/deserializer machinery[^1]. If you have called `mapper.readValue(...)` or `writeValueAsString(...)` in a Java service, you used this repo.

Its 3.7k stars understate its reach: databind is infrastructure, pulled in transitively by Spring Boot, Elasticsearch, Kafka, the AWS SDK, and thousands of other libraries rather than starred directly. It is the default JSON binding in Spring Boot and Jakarta REST implementations. The 1.5k forks and steady commit cadence (last push July 2026) reflect an actively maintained project still led by original author Tatu Saloranta[^2].

The defining tension is **flexibility versus safety**. Databind's willingness to instantiate arbitrary Java types from untrusted input — the same power that makes polymorphic mapping convenient — produced a long run of remote-code-execution CVEs in the 2017–2019 era. Much of the library's later evolution is a story of walking that back with opt-in validators and hard parser limits.

## Getting Started

Maven (Jackson 3.x — note the new `tools.jackson.core` groupId):

```xml
<dependency>
  <groupId>tools.jackson.core</groupId>
  <artifactId>jackson-databind</artifactId>
  <version>3.0.0</version>
</dependency>
```

For the still-widely-used 2.x line the coordinates are `com.fasterxml.jackson.core:jackson-databind`. Basic round-trip:

```java
// tools.jackson.databind.* in 3.x; com.fasterxml.jackson.databind.* in 2.x
record MyValue(String name, int age) {}

ObjectMapper mapper = new ObjectMapper(); // construct once, reuse everywhere

MyValue v = mapper.readValue("{\"name\":\"Bob\",\"age\":13}", MyValue.class);
String json = mapper.writeValueAsString(v);  // {"name":"Bob","age":13}

// Tree model when structure is dynamic or only partially known:
JsonNode root = mapper.readTree(json);
String name = root.get("name").asText();
```

## Architecture / How It Works

Three processing models coexist, layered on the same streaming core:

1. **Streaming** (`JsonParser` / `JsonGenerator`, from `jackson-core`) — token-at-a-time, lowest overhead, most verbose. Both models below are built on it.
2. **Tree model** — parse into an in-memory `JsonNode` graph (`ObjectNode`, `ArrayNode`, value nodes). Convenient for dynamic or heterogeneous documents; heavier than data-binding for known shapes.
3. **Data-binding** — the headline feature. `ObjectMapper` introspects a target type via reflection, builds and caches a `BeanSerializer` / `BeanDeserializer`, then maps fields, getters/setters, and constructors to JSON properties. `@JsonCreator`, `@JsonProperty`, `@JsonIgnore`, and mix-ins tune the mapping.

`ObjectMapper` is the central object. It is expensive to build (reflection, module registration) and **thread-safe once configured** — the intended pattern is one shared instance per application. In 2.x it was mutable via `configure(...)` calls; 3.x makes it immutable and forces the `JsonMapper.builder()...build()` pattern so a fully-constructed mapper can never be reconfigured under concurrent use[^3].

Behavior beyond core JSON lives in **modules** you register explicitly: `JavaTimeModule` (jsr310) for `java.time`, `jackson-module-kotlin` for Kotlin data classes, `jackson-datatype-jdk8` for `Optional`, plus the `jackson-dataformat-*` family (XML, YAML, CSV, CBOR, Smile, Avro, Protobuf) that swaps the streaming backend so the same binding code targets non-JSON formats. Databind itself has no hard dependency on JSON — the "JSON" naming is historical.

## Production Notes

**ObjectMapper reuse.** Constructing a mapper per request is the most common performance bug — it discards the serializer cache and repeats reflection every call. Build one, share it. Need per-call tweaks? Use `mapper.reader()` / `mapper.writer()` views, which are cheap and thread-safe.

**Version alignment.** `jackson-databind`, `jackson-core`, and `jackson-annotations` must be on compatible versions. A mismatched transitive pull (very common in large dependency trees) surfaces as `NoSuchMethodError` or `NoClassDefFoundError` at runtime, not at build time. Import `jackson-bom` (or rely on Spring Boot's managed versions) to pin them together.

**Polymorphic deserialization / the CVE lineage.** The historical footgun. `enableDefaultTyping()` plus a `@JsonTypeInfo` field let attackers name a gadget class in the JSON and trigger RCE; this drove dozens of CVEs. Jackson 2.10 removed the unrestricted API and replaced it with `activateDefaultTyping(PolymorphicTypeValidator)`, forcing an allow-list of permitted base types[^4]. Never enable default typing against untrusted input without a tight validator.

**DoS via malicious input.** Deeply nested or gigantic documents could exhaust memory/stack. Jackson 2.15 added `StreamReadConstraints` — default caps on nesting depth, number length, and string length — enforced in the parser[^5]. Raise them deliberately if you genuinely process large documents; do not disable them on public endpoints.

**Unknown properties fail by default.** `FAIL_ON_UNKNOWN_PROPERTIES` is on, so a new field on the producer side breaks consumers. Most services disable it (`DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES`) for forward compatibility.

**`java.time` needs a module.** Without `JavaTimeModule` registered, `LocalDate`/`Instant` serialize as ugly numeric arrays or throw. Register it and usually disable `WRITE_DATES_AS_TIMESTAMPS` for ISO-8601 output.

**GraalVM native image.** Reflection-driven binding needs reachability metadata; unconfigured types fail at native runtime. Spring Boot supplies hints; standalone native apps must provide reflection config or use the Jackson native hints.

**2.x → 3.x is a major migration.** New `tools.jackson` groupId and package names, JDK 17 baseline (2.x targets JDK 8), immutable mapper, and dropped deprecated APIs. It is a coordinated ecosystem cutover, not a drop-in upgrade — expect to touch imports and mapper-construction code[^6].

## When to Use / When Not

**Use when:**
- You are on the JVM and need JSON (or XML/YAML/CBOR) binding — it is the ecosystem default and integrates with virtually every framework.
- You need annotation-driven control over naming, ignoring, polymorphism, or custom (de)serializers.
- You want one API spanning streaming, tree, and full data-binding.

**Avoid / reconsider when:**
- You want zero reflection and minimal startup (GraalVM-heavy, serverless cold-start sensitive) — a compile-time binder may fit better.
- You deserialize untrusted, attacker-controlled JSON into polymorphic types — the attack surface demands careful lockdown; a stricter binder reduces risk.
- You only need trivial config-file parsing — the full dependency (three jars + modules) may be more than warranted.

## Alternatives

- google/gson — simpler, reflection-based JSON binder; fewer features and modules, gentler learning curve. Use when you want lightweight JSON without Jackson's surface area.
- square/moshi — Kotlin-first with codegen adapters that avoid reflection. Use on Android/Kotlin where startup and R8 friendliness matter.
- alibaba/fastjson — very fast, but carries its own serious CVE history around auto-type; fastjson2 is the reworked successor. Use only with autotype disabled and eyes open.
- FasterXML/jackson-core — drop to this when you only need streaming parse/generate and want none of the binding overhead.
- protocolbuffers/protobuf — when you control both ends and want a schema-driven binary format instead of JSON.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2009 | Original Jackson under `org.codehaus.jackson`, by Tatu Saloranta[^2]. |
| 2.0 | 2012 | Rewrite under `com.fasterxml.jackson`; databind split into its own module. |
| 2.10 | 2019-09 | Safe default typing (`PolymorphicTypeValidator`); JPMS `module-info`[^4]. |
| 2.12 | 2020-11 | Gradle Module Metadata; many binding refinements. |
| 2.15 | 2023-04 | `StreamReadConstraints` DoS limits enabled by default[^5]. |
| 3.0 | 2025–2026 | New `tools.jackson` coordinates, JDK 17 baseline, immutable `ObjectMapper`[^6]. |

## References

[^1]: jackson-databind README, "Overview" — data-binding builds on jackson-core streaming and jackson-annotations. https://github.com/FasterXML/jackson-databind
[^2]: Tatu Saloranta (cowtowncoder), Jackson original author. https://github.com/cowtowncoder
[^3]: jackson-databind README, "10 minute tutorial: configuration" — 3.x requires builder-style construction to keep `ObjectMapper` immutable and thread-safe.
[^4]: Jackson 2.10 release notes — default typing safety and JPMS. https://github.com/FasterXML/jackson/wiki/Jackson-Release-2.10
[^5]: Jackson 2.15 release notes — StreamReadConstraints. https://github.com/FasterXML/jackson/wiki/Jackson-Release-2.15
[^6]: Jackson 3.0 migration — coordinates, JDK 17, immutable mapper. https://github.com/FasterXML/jackson-databind/blob/3.x/README.md

## Tags

java, json, serialization, data-binding, jvm, object-mapper, deserialization, jackson, xml, library
