# mapstruct/mapstruct

> A Java annotation processor that generates type-safe bean-mapping code at compile time — no reflection, no runtime dependency.

[GitHub repo](https://github.com/mapstruct/mapstruct) ·
[Official website](https://mapstruct.org/) ·
[License: Apache-2.0](https://github.com/mapstruct/mapstruct/blob/main/LICENSE.txt)

## Overview

MapStruct solves a narrow, ubiquitous Java problem: copying values between two
object shapes — typically a JPA entity and a DTO, or a domain object and an API
model. Instead of hand-writing `dto.setName(entity.getName())` a hundred times,
or reaching for a reflection-based mapper, you declare an interface with
`@Mapper` and MapStruct generates a plain-Java implementation during
compilation via JSR 269 annotation processing[^1]. The generated code is
ordinary getter/setter calls you can open, read, and step through in a
debugger.

The defining tradeoff is *compile-time over runtime*. Because mapping is
resolved by the annotation processor, a missing or ambiguous mapping is a build
error, not a production surprise, and there is zero reflection cost at runtime.
The price is that MapStruct lives inside your build: it is sensitive to
annotation-processor ordering (the Lombok interaction is the canonical
footgun), to incremental-compilation staleness, and to IDE tooling that must be
configured to show generated sources. It is a mature, conservatively versioned
project — first released 2016, active since 2012 — and the de facto standard
for bean mapping in the Spring ecosystem[^2]. Current stable is 1.6.3[^3].

## Getting Started

Two artifacts: `mapstruct` (annotations, a compile dependency) and
`mapstruct-processor` (the annotation processor, on the processor path only).

```xml
<properties><org.mapstruct.version>1.6.3</org.mapstruct.version></properties>
<dependencies>
  <dependency>
    <groupId>org.mapstruct</groupId>
    <artifactId>mapstruct</artifactId>
    <version>${org.mapstruct.version}</version>
  </dependency>
</dependencies>
<!-- add mapstruct-processor to maven-compiler-plugin <annotationProcessorPaths> -->
```

```java
@Mapper(componentModel = "spring")   // generates an injectable @Component
public interface CarMapper {

    @Mapping(target = "seatCount", source = "numberOfSeats")
    CarDto carToCarDto(Car car);
}
```

At compile time MapStruct writes `CarMapperImpl` into
`target/generated-sources/annotations`. Properties with matching names map
automatically; `@Mapping` handles renames, nested paths, constants,
expressions, and formatting.

## Architecture / How It Works

MapStruct is a `javax.annotation.processing.Processor` that runs inside `javac`
(or the Gradle/Maven compiler task, Kotlin's `kapt`, etc.). For each `@Mapper`
type it walks the source and target classes via the `javax.lang.model` mirror
API — reading getters, setters, constructors, builders, and record
components — and emits a concrete implementation class as text. There is no
bytecode manipulation and nothing happens at application startup.

Mapping resolution is a set of ordered strategies: same-name property matching,
built-in type conversions (primitives, wrappers, `String`↔number/date,
`java.time`, enums), then user-provided mapping methods discovered by
signature. When two source objects feed one target you add parameters; when you
want to update an existing instance you annotate a parameter `@MappingTarget`.
`@InheritConfiguration` and `@InheritInverseConfiguration` reuse a mapping's
rules for its reverse or an update variant. `@Named` + `qualifiedByName`
disambiguate when several candidate methods could satisfy a mapping.

`componentModel` decides how the generated mapper is instantiated: `default`
(a static `Mappers.getMapper()` factory), `spring` (a `@Component`), `jsr330`,
`jakarta`, or CDI — this is why the same annotations drop cleanly into Spring
Boot, Quarkus, or Jakarta EE. Everything the processor knows is visible in the
generated `*Impl.java`, which is the intended debugging surface: if the mapper
misbehaves, you read the generated code, not the library internals.

## Production Notes

**Lombok ordering is the number-one issue.** Lombok generates getters/setters
in the same annotation-processing round MapStruct reads them. If MapStruct runs
first it sees a bean with no accessors and generates empty mappings. The fix is
the `lombok-mapstruct-binding` artifact plus correct processor ordering on the
path (Lombok before MapStruct)[^4]. Silent empty mappings from a missing
binding are a recurring support thread.

**`unmappedTargetPolicy` defaults to `WARN`, not `ERROR`.** Add a new field to a
DTO and MapStruct silently leaves it null while emitting a build warning most
CI logs bury. Teams that care about mapping completeness set
`@Mapper(unmappedTargetPolicy = ReportingPolicy.ERROR)` (or via `MapperConfig`)
so a forgotten field fails the build.

**Incremental compilation can serve stale mappers.** Gradle's incremental
annotation processing and IDE build caches sometimes keep an old `*Impl` after
you change a mapping; a clean rebuild is the reliable reset. Similarly, IntelliJ
and Eclipse show generated code only after annotation processing is enabled and
a build has run — the mapper "not existing" in the editor is almost always a
tooling-config problem, not a code problem.

**Expressions and `imports` are stringly-typed.** `expression = "java(...)"`
and `defaultExpression` embed raw Java in an annotation string; they are not
checked until the generated code compiles, and refactors/renames do not reach
inside them. Prefer real mapping methods over expressions where practical.

**Runtime footprint is genuinely zero.** The `mapstruct` annotations are the
only thing shipped, and generated code has no MapStruct runtime dependency —
attractive for libraries, native images (GraalVM), and reflection-averse
deployments.

## When to Use / When Not

**Use when:**
- You have many entity↔DTO shapes and want mapping errors at build time.
- You are on Spring Boot / Jakarta and want injectable mappers with no glue.
- You need reflection-free mapping (GraalVM native image, tight latency).
- You want to read and debug the mapping code that actually runs.

**Avoid when:**
- Shapes are highly dynamic or unknown until runtime — a reflective mapper like
  ModelMapper fits better than a compile-time processor.
- You cannot tolerate annotation-processor complexity in your build (kapt,
  Lombok ordering, incremental-build quirks).
- The mapping is one or two classes — hand-written code is simpler than adding a
  processor.

## Alternatives

- modelmapper/modelmapper — use instead when mappings must be resolved at
  runtime by convention and you accept reflection cost for flexibility.
- jhalterman/... Orika — reflection/bytecode mapper; use when you need runtime
  configuration but faster-than-ModelMapper throughput (largely unmaintained).
- DozerMapper/dozer — legacy XML/reflection mapper; effectively deprecated, use
  only for maintaining existing Dozer code.
- immutables/immutables — overlaps for value objects; its own processor can
  generate builders MapStruct then targets, rather than competing directly.
- Hand-written mappers — for a handful of types, plain getter/setter code has
  zero build complexity and is the honest baseline MapStruct competes against.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0.Final | 2016 | First stable release; core `@Mapper`/`@Mapping` model[^1]. |
| 1.2.0.Final | 2017 | Adder methods, stream mapping, decorators. |
| 1.3.0.Final | 2019 | Jakarta/CDI improvements, presence checks. |
| 1.4.0.Final | 2020 | Builder support, constructor mapping foundations. |
| 1.5.0.Final | 2022 | Java 16+ record support, `@SubclassMapping`, Spring meta-annotations[^5]. |
| 1.6.0.Final | 2024 | Conditional mapping expressions, enum/subclass refinements[^6]. |
| 1.6.3 | 2025 | Current stable maintenance release[^3]. |

## References

[^1]: MapStruct README — "What is MapStruct?" and annotation-processor overview. https://github.com/mapstruct/mapstruct
[^2]: MapStruct reference guide, component models (Spring/CDI/JSR-330). https://mapstruct.org/documentation/reference-guide/
[^3]: Latest stable version badge (1.6.3), README. https://central.sonatype.com/search?q=g:org.mapstruct
[^4]: MapStruct docs, "Lombok" — annotation-processor ordering and `lombok-mapstruct-binding`. https://mapstruct.org/documentation/stable/reference/html/#lombok
[^5]: MapStruct 1.5 release notes — records and `@SubclassMapping`. https://github.com/mapstruct/mapstruct/releases
[^6]: MapStruct 1.6 release notes — conditional expressions and refinements. https://github.com/mapstruct/mapstruct/releases

## Tags

java, annotation-processor, bean-mapping, dto, code-generation, compile-time, spring, jvm, mapper, jsr-269
