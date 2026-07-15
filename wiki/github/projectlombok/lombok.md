# projectlombok/lombok

> A Java annotation library that rewrites your class during compilation to erase getter/setter/equals/builder boilerplate — by reaching into the compiler's private internals.

[GitHub repo](https://github.com/projectlombok/lombok) ·
[Official website](https://projectlombok.org/) ·
[License: MIT](https://github.com/projectlombok/lombok/blob/master/LICENSE)

## Overview

Lombok is a compile-time code generator for Java, first released in 2009 by
Reinier Zwitserloot and Roel Spilker[^1]. You annotate a class with `@Getter`,
`@Data`, `@Builder`, `@Slf4j` and so on, and Lombok injects the corresponding
methods, fields, and constructors so they never appear in your source. It is one
of the most widely deployed Java libraries — pulled in transitively by a large
share of Spring Boot projects — and one of the most divisive.

The division comes from *how* it works. Standard Java annotation processors can
only generate new files; they cannot modify the class being compiled. Lombok
does modify it: it hooks into `javac`'s internal abstract syntax tree via the
non-public `com.sun.tools.javac.*` API and mutates the AST of your class in
place during compilation[^2]. This is explicitly outside the contract the JDK
offers annotation processors, which is why every major JDK release carries a
risk of breaking Lombok until a compatibility patch ships. The payoff is real —
it erases a category of boilerplate the language spent a decade refusing to
address — but the price is a hard dependency on unpublished compiler internals,
IDE plugins, and a build that can break on a JDK upgrade. Since Java 16 shipped
`record` types, part of Lombok's original justification (immutable data
carriers) is now covered by the language itself.

## Getting Started

Maven:

```xml
<dependency>
  <groupId>org.projectlombok</groupId>
  <artifactId>lombok</artifactId>
  <version>1.18.34</version>
  <scope>provided</scope>
</dependency>
```

Gradle:

```groovy
compileOnly 'org.projectlombok:lombok:1.18.34'
annotationProcessor 'org.projectlombok:lombok:1.18.34'
```

```java
import lombok.Data;
import lombok.Builder;

@Data                 // getters, setters, toString, equals, hashCode
@Builder              // fluent builder: User.builder().name("x").build()
public class User {
    private final String name;
    private int age;
}
```

The scope must be `provided` / `compileOnly`: Lombok is a build-time tool and
should not ship in your runtime classpath. IntelliJ IDEA bundles Lombok support
since 2020.3; Eclipse and VS Code require the Lombok agent or extension, or the
IDE flags the generated members as errors.

## Architecture / How It Works

Lombok runs as an annotation processor, but that is a foothold, not the
mechanism. Once `javac` invokes it, Lombok obtains a handle to the compiler's
live parse tree and edits it directly — adding method nodes, constructor nodes,
and field references before the bytecode-generation phase runs[^2]. The
generated members therefore exist in the `.class` output but never in any
`.java` file.

Because it depends on compiler internals rather than a stable SPI, Lombok
maintains separate integration paths per toolchain:

- **javac** — AST manipulation through `com.sun.tools.javac` internals.
- **Eclipse / ecj** — installed as a `-javaagent` that patches the Eclipse
  compiler's AST classes at class-load time[^3], which is why Eclipse users run
  the Lombok installer rather than just adding a dependency.
- **IDE indexing** — IntelliJ and VS Code need a plugin so the editor's model of
  the class includes the synthesized members; without it, a call to `getName()`
  shows as an unresolved symbol even though it compiles.

Configuration is file-based via `lombok.config`, discovered hierarchically up
the directory tree (chained setters, `@NonNull` null-check style, and so on). A
companion tool, **delombok**, expands all Lombok annotations back into plain Java
source — useful for readable output, feeding other processors, or exiting the
dependency. Key annotations cluster into accessors (`@Getter`/`@Setter`), value
semantics (`@EqualsAndHashCode`, `@ToString`, `@Data`, `@Value`), construction
(`@Builder`, `@SuperBuilder`, the `*Constructor` family), immutability helpers
(`@With`), logging (`@Slf4j`, `@Log`), and control-flow sugar (`@SneakyThrows`,
`@Cleanup`, `@Synchronized`).

## Production Notes

**JDK upgrades are the recurring footgun.** Because Lombok reaches into
`jdk.compiler` internals, each major JDK release has historically had a window
where the current Lombok does not compile until a patch ships. JDK 9's module
system and the strong encapsulation from JDK 16 onward (JEP 396/403) both forced
workarounds, and teams on early-access or brand-new releases regularly hit
"Lombok can't process compilation unit" failures. Bump Lombok deliberately when
you move JDKs; do not assume an old Lombok works on a new compiler.

**Annotation-processor ordering with MapStruct.** MapStruct can process a class
before Lombok has generated the accessors it needs, producing mappers that
reference missing getters/setters. The fix is the `lombok-mapstruct-binding`
artifact, which enforces ordering[^4] — a common first-time integration failure.

**Adopting is easy; leaving is not.** The boilerplate Lombok hides no longer
exists in your source, so exit requires `delombok` to materialize the generated
code back into real `.java` files — mechanical output that usually needs cleanup.

**Debugging and tooling friction.** Stack traces point at synthesized methods
that have no source lines, and some coverage, static-analysis, and refactoring
tools misbehave on Lombok-processed classes. `@SneakyThrows` in particular hides
checked exceptions from the type system, which can surprise callers.

**Records overlap.** On JDK 16+, `record` covers immutable data carriers
natively with no dependency. Lombok still wins for mutable entities, builders on
large classes, and JPA `@Entity` types (which cannot be records) — but new
immutable value types are often better as plain records.

## When to Use / When Not

**Use when:**
- You have many DTOs, JPA entities, or config classes drowning in
  getter/setter/equals/builder boilerplate.
- Your team already standardizes on it and has the IDE plugins installed.
- You want `@Builder` / `@Slf4j` ergonomics without hand-writing them.

**Avoid when:**
- You are on bleeding-edge or frequently-changing JDKs and cannot tolerate a
  build broken by a compiler-internals change.
- Your data types are immutable and JDK 16+ `record` would suffice.
- You are writing a widely-consumed library — forcing an AST-hacking build-time
  dependency and IDE plugins onto downstream users is a real cost.

## Alternatives

- immutables/immutables — standards-compliant annotation processor that
  *generates* immutable value classes as separate files; use when you want no
  compiler-internals hacking and prefer explicit generated code.
- google/auto — AutoValue generates value-type implementations via the public
  APT contract; use for Google-maintained, spec-abiding codegen for value types.
- JDK `record` (Java 16+) — use for immutable data carriers where a language
  feature beats any dependency.
- JetBrains/kotlin — `data class`, properties, and null-safety solve the same
  boilerplate at the language level; use when you can adopt Kotlin.
- IDE "Generate" actions — for a small class count, generated-into-source
  accessors avoid every Lombok tradeoff at the price of verbosity.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.9.x | 2009–2010 | Initial public releases; core accessor annotations[^1]. |
| 0.11 / 1.12 | 2013 | `@Builder`, `@Value`, `@Slf4j` and logging family stabilized. |
| 1.18.0 | 2018-06 | JDK 10/11 support; module-system compatibility work. |
| 1.18.20 | 2021-03 | JDK 16 strong-encapsulation compatibility, `record` support. |
| 1.18.34 | 2024 | Ongoing JDK 21/22 compatibility line. |

## References

[^1]: Project Lombok — official site and feature overview. https://projectlombok.org/features/
[^2]: Project Lombok internals: annotation-based AST manipulation of the javac compilation unit. https://projectlombok.org/contributing/lombok-execution-path
[^3]: Lombok Eclipse/ecj integration via a Java agent that patches the compiler at load time. https://projectlombok.org/setup/eclipse
[^4]: `lombok-mapstruct-binding` — enforces annotation-processor ordering between Lombok and MapStruct. https://github.com/mapstruct/mapstruct/issues/1581

## Tags

java, jvm, annotation-processing, code-generation, boilerplate, compiler, ast-manipulation, developer-tools, build-tool, javac
