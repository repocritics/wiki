# immutables/immutables

> Java annotation processor that generates immutable value classes, builders, and with-ers from a hand-written abstract type — records included.

[GitHub repo](https://github.com/immutables/immutables) ·
[Official website](http://immutables.org) ·
[License: Apache-2.0](https://github.com/immutables/immutables/blob/master/LICENSE)

## Overview

Immutables is a compile-time code generator for the JVM. You declare an abstract type — an interface, abstract class, or (since Java 14 records) a record — annotate it with `@Value.Immutable`, and the annotation processor emits a concrete `Immutable*` implementation with a builder, value-based `equals`/`hashCode`/`toString`, null checks, and defensive collection copies[^1]. The project dates to 2013 and predates both Java records and most of its current competitors; it is one of the older, more feature-dense entries in the Java "kill the boilerplate" space.

Its defining characteristic is that it generates *ordinary Java source* at compile time rather than rewriting bytecode. This is the central tradeoff versus Lombok: Immutables produces readable generated classes you can open, debug, and step through, and it needs no runtime dependency and no IDE plugin to *run* — but it forces you into a two-type world (the abstract `Foo` you write, the generated `ImmutableFoo` your callers construct) and pays the full cost of annotation processing on every build. It is aimed at teams that want strict, verifiable immutability and are willing to accept generated-type coupling and build-time overhead to get it.

The library is broad to the point of sprawl. Beyond the core value objects it ships a heavily configurable `@Value.Style` system, Jackson/Gson serialization integration, optional Guava collection support, a custom-attribute `@Encoding` meta-facility, and a separate Criteria module for type-safe queries against document stores. Most projects use a thin slice of this surface.

## Getting Started

Maven — the processor is only needed at compile time:

```xml
<dependency>
  <groupId>org.immutables</groupId>
  <artifactId>value</artifactId>
  <version>2.10.1</version>
  <scope>provided</scope>
</dependency>
```

Gradle:

```groovy
dependencies {
  compileOnly     "org.immutables:value:2.10.1"
  annotationProcessor "org.immutables:value:2.10.1"
}
```

```java
import org.immutables.value.Value;
import java.util.List;
import java.util.Optional;

@Value.Immutable
interface Book {
  String isbn();
  String title();
  List<String> authors();          // generated builder gets addAuthors(...)
  Optional<String> subtitle();     // Optional attributes are optional in the builder
}

// ImmutableBook is generated at compile time
ImmutableBook book = ImmutableBook.builder()
    .isbn("978-1-56619-909-4")
    .title("The Elements of Style")
    .addAuthors("Strunk", "White")
    .build();
```

## Architecture / How It Works

Immutables is a standard JSR-269 annotation processor. During `javac`'s annotation-processing round it reads the abstract type's accessor methods and generates a sibling class in the same package. Accessor names define attributes; return types define builder methods (a `List<T>` accessor yields `addX`/`addAllX`, a `Map` yields `putX`, an `Optional` becomes optional in the builder).

Immutability is enforced structurally: the generated class is `final`, fields are `final` and `private`, collection attributes are copied into unmodifiable implementations at build time, and constructor arguments are null-checked unless annotated `@Nullable`. Attribute semantics are controlled by companion annotations — `@Value.Default` (default-valued methods), `@Value.Derived` (computed once at construction), `@Value.Lazy` (computed on first access, memoized), `@Value.Parameter` (positional constructor `of(...)` args), and `@Value.Check` (post-construction validation/normalization).

The most consequential subsystem is `@Value.Style`. It reprograms nearly everything about the output: the generated class name pattern (the `Immutable*` prefix is only a default), builder method prefixes, whether to emit `with*` copy methods, strict vs. lenient builders, visibility, staged/telescoping builders that make required attributes a compile-time sequence, and more. Styles can be attached to a type, a package (`package-info.java`), or a custom meta-annotation. This makes the library extremely malleable and also makes two Immutables codebases look nothing alike.

Serialization is opt-in and integration-specific. With `@JsonSerialize`/`@JsonDeserialize` pointing at the immutable/builder, Jackson round-trips through the generated builder. Gson support generates `TypeAdapter`s registered via a `TypeAdapterFactory`. Guava is used automatically for collection attributes *if it is on the classpath*, otherwise the processor falls back to JDK collections — a quiet behavioral fork worth knowing about.

The separate **Criteria** module builds on the same code generation to produce a type-safe, fluent query DSL plus repository abstractions with backends for document stores. It is a distinct product with its own maturity curve and is unrelated to the core value-object use case.

## Production Notes

- **Generated-type coupling is real and viral.** Callers construct and reference `ImmutableFoo`, not `Foo`. The common `class Builder extends ImmutableFoo.Builder {}` "sandwich" pattern and `With*` interfaces exist largely to hide the generated names behind your own — plan the pattern before the types spread across a codebase, because retrofitting it later is a wide edit.
- **Annotation processing must be wired in every build path.** IntelliJ and Eclipse need annotation processing enabled and the `generated-sources` directory marked as a source root, or you get red squiggles on `ImmutableFoo` that compile fine on the CLI. Gradle needs the `annotationProcessor` configuration (not just `compileOnly`); mixing them up produces classes-not-found only at build time.
- **Build-time cost scales with count.** Hundreds of value types generate hundreds of classes each round; on large modules Immutables is a measurable slice of compile time. Incremental annotation processing helps but is sensitive to how styles and `package-info` styles are structured.
- **Don't mix with Lombok on the same type.** Both are annotation processors touching construction/accessors; ordering and duplicate-member conflicts are a known source of nondeterministic build failures.
- **Records reduce but don't remove the case for it.** Java records give you the transparent carrier, `equals`/`hashCode`, and accessors for free. Immutables still adds builders, `with*` copy methods, staged builders, defaults/derived/lazy/check, and collection ergonomics that records lack — which is why record support (`@Value.Builder` on a record, `With*` interfaces) was added rather than the project being retired.
- **No runtime dependency, but check retention if you reflect.** The value annotations are not needed at runtime for the generated code to work; teams sometimes assume they are available for reflection and find they are not on the runtime classpath.
- **Open-issue count is high (400+).** For a project of this age and scope, treat the tracker as a backlog of edge cases and style requests rather than a signal of instability; the core has been stable for years and releases continue.

## When to Use / When Not

**Use when:**
- You want compiler-enforced, genuinely immutable value objects with builders and `with*` copies, and you value readable generated source over bytecode magic.
- You need rich construction semantics — staged/required builders, defaults, derived and lazy attributes, `@Value.Check` normalization — that records and plain classes don't provide.
- You want zero runtime dependency and no IDE plugin requirement, and can absorb annotation-processor setup.

**Avoid when:**
- Plain Java records (optionally plus a small builder generator) already cover your needs — pulling in Immutables' surface area is overkill.
- Your team wants minimal ceremony and is comfortable with bytecode-rewriting tools; Lombok's `@Value`/`@Builder` are terser.
- You can't tolerate the two-type (`Foo`/`ImmutableFoo`) coupling or the added build-time cost across many modules.

## Alternatives

- google/auto (AutoValue) — use when you want a smaller, more conservative Google-maintained value-type generator with far fewer knobs than `@Value.Style`.
- projectlombok/lombok — use when you prefer terse bytecode-based `@Value`/`@Builder` and accept its IDE-plugin and agent-based tradeoffs.
- Java records (JDK built-in) — use when you only need transparent immutable carriers and can forgo builders, with-ers, and validation.
- randgalt/record-builder — use when you already have records and want only builders and `with*` methods, without Immutables' breadth.
- Kotlin data classes — use when adopting Kotlin is on the table; `copy()` and value semantics are language-level.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2013-07 | Repository created; early `org.immutables` value generation[^2]. |
| 2.0 | 2015 | Major line; consolidated under `org.immutables:value`, expanded `@Value.Style`[^3]. |
| 2.x | 2016–2023 | Long-lived stable series: Jackson/Gson integration, encodings, Criteria module. |
| 2.9–2.10 | 2023–2024 | Java records support, builder-on-record, continued 2.x maintenance[^3]. |

Exact per-release dates and notes are on the GitHub releases page; earlier entries are in the archived changelog[^2][^3].

## References

[^1]: Immutables README and documentation — `@Value.Immutable`, generated builders, record support. https://github.com/immutables/immutables and http://immutables.org
[^2]: immutables/immutables repository — created 2013-07-07; archived changelog under `.archive/CHANGELOG.md`. https://github.com/immutables/immutables
[^3]: Immutables releases (2.x series, records support). https://github.com/immutables/immutables/releases

## Tags

java, annotation-processor, code-generation, immutable-objects, builder-pattern, value-types, jvm, records, jsr-269, serialization
