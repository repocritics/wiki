# vavr-io/vavr

> An object-functional library for Java 8+: persistent collections plus Option/Try/Either and pattern matching, with no runtime dependencies.

[GitHub repo](https://github.com/vavr-io/vavr) ·
[Official website](https://vavr.io) ·
[License: Apache-2.0](https://github.com/vavr-io/vavr/blob/main/LICENSE)

## Overview

Vavr (pronounced "vavr", stylized "vʌvr") is a Java library that ports ideas from functional languages — persistent immutable collections, algebraic control types, tuples, function objects, and a pattern-matching DSL — onto the Java type system. It targets Java 8+ and has no dependencies beyond the JDK, so it drops in as a single jar[^1].

The project was originally released as **Javaslang** and renamed to **Vavr** at the 0.9.0 release (2017), reportedly to avoid the "Java" trademark association[^2]. It was created by Daniel Dietrich; it is now led and maintained by Grzegorz Piwowarek (@pivovarit)[^1]. The core value proposition is "defensive programming made easy": replace nulls with `Option`, replace thrown checked exceptions with `Try`, and replace mutable `java.util` collections with structural-sharing persistent ones.

The defining tension is that the Java standard library has moved toward Vavr's territory. `Optional`, `Stream`, records, sealed types, and `switch` pattern matching (Java 21) cover a growing fraction of what Vavr offered when it launched in 2014. Vavr remains richer and more consistent than the JDK equivalents, but the case for adopting a whole parallel type universe is narrower on modern Java than it was on Java 8. Compounding this, the library spent years between the 0.10.x line and the 1.0.0 release with low visible velocity, which made some teams wary of it as a core dependency.

## Getting Started

Maven:

```xml
<dependency>
    <groupId>io.vavr</groupId>
    <artifactId>vavr</artifactId>
    <version>1.0.0</version>
</dependency>
```

Gradle:

```groovy
implementation 'io.vavr:vavr:1.0.0'
```

```java
import io.vavr.collection.List;
import io.vavr.control.Option;
import io.vavr.control.Try;

// Persistent, immutable list — map returns a new list, original is untouched.
List<Integer> squares = List.of(1, 2, 3, 4).map(n -> n * n); // List(1, 4, 9, 16)

// Option replaces null; headOption never throws on an empty list.
Option<Integer> first = squares.headOption();                // Some(1)

// Try captures a throwing computation as a value instead of propagating it.
int parsed = Try.of(() -> Integer.parseInt("42"))
    .map(n -> n + 1)
    .getOrElse(-1);                                          // 43
```

Pattern matching uses a static-import DSL rather than a language feature:

```java
import static io.vavr.API.*;
import static io.vavr.Predicates.*;

String label = Match(input).of(
    Case($(0), "zero"),
    Case($(isIn(1, 2, 3)), "small"),
    Case($(), "other")
);
```

## Architecture / How It Works

Vavr ships as a small set of coordinated pieces. The `io.vavr` core provides:

- **Control types** — `Option`, `Try`, `Either`, `Validation` (accumulating errors), and `Lazy` (memoized deferred value). These are Vavr's own types, not adapters over `java.util.Optional`.
- **Persistent collections** — `List` (linked), `Stream` (lazy linked list), `Array`, `Vector`, `Queue`, plus `HashMap`/`TreeMap`, `HashSet`/`TreeSet`, `LinkedHashMap`, and multimaps. `HashMap`/`HashSet` are backed by a HAMT (hash array mapped trie); the sequential structures use structural sharing so that "modification" returns a new value cheaply.
- **Tuples** — `Tuple0` through `Tuple8`, first-class fixed-arity heterogeneous values.
- **Functions** — `Function0`–`Function8` and `CheckedFunction0`–`8`, with currying, partial application, memoization, and composition.

Two companion modules exist: `vavr-match` (an annotation processor that generates `@Patterns` deconstructors for user types) and `vavr-test` (property-based / generator testing). All of it is pure library code — there is no runtime, agent, or bytecode weaving.

The pattern-matching API is worth understanding because it is not a language construct: `Match`, `Case`, and `$()` are ordinary static methods returning objects. This makes it work on any Java 8 compiler, but it is more verbose than Java 21's native `switch` patterns and cannot exhaustively check sealed hierarchies at compile time.

The interop boundary is explicit and pervasive: Vavr collections are not `java.util.Collection` subtypes in spirit even where interfaces overlap, so crossing into stream/framework code that expects JDK collections means calling `.toJavaList()`, `.toJavaMap()`, `.asJava()`, and friends. Vavr's `Stream` is a lazy linked list and is unrelated to `java.util.stream.Stream` despite the shared name — a frequent source of confusion.

## Production Notes

**Maintenance cadence.** The 0.10.x line (0.10.0 in 2019, patch releases through the early 2020s) remained the de facto stable version for a long stretch while 1.0.0 sat in alpha. Velocity has picked up under the current maintainer, but anyone adopting Vavr as a foundational dependency should look at recent commit and release activity for themselves rather than assume a fast-moving project.

**Overlap with modern Java.** On Java 17/21, `Optional`, records, sealed interfaces, and `switch` pattern matching subsume the most common reasons people reached for Vavr. Introducing Vavr into a codebase that already uses these means two idioms for the same concept (`Option` vs `Optional`, Vavr `Match` vs native switch). Pick one boundary and convert at it, rather than mixing throughout.

**Allocation and boxing overhead.** Persistent collections trade mutation for allocation: every "update" allocates nodes. For hot loops over primitives this is measurably slower and more GC-heavy than a mutable `ArrayList` or a specialized primitive collection. `Option`/`Try` also box. Vavr is a correctness and expressiveness tool, not a performance optimization; benchmark before using it on a critical path.

**Interop friction.** Serialization frameworks (Jackson, JPA, Bean Validation) do not understand Vavr types out of the box. Jackson needs the community `vavr-jackson` module; persistence layers generally require converting to JDK types at the boundary. Budget for adapter code at every framework edge.

**Name collisions.** `io.vavr.collection.List`/`Stream`/`Map` shadow `java.util` names. Mixed imports are a readability hazard; teams often ban wildcard imports and alias deliberately.

**Upgrade note.** Migrating from Javaslang (`javaslang.*` packages, pre-0.9) to Vavr (`io.vavr.*`) is a package rename plus coordinate change (`io.javaslang:javaslang` → `io.vavr:vavr`); it is mechanical but touches every import site.

## When to Use / When Not

**Use when:**
- You are on Java 8–11 and want `Option`, `Try`, and immutable collections that the JDK of that era lacks.
- You want accumulating validation (`Validation`), rich `Either`, or property-based testing in a single dependency-free jar.
- Your team prefers a consistent functional style across control flow and collections rather than the JDK's piecemeal additions.

**Avoid when:**
- You are on Java 21+ and the standard library already covers your needs — records, sealed types, and switch patterns reduce the marginal value.
- You are performance-sensitive on hot paths where persistent-collection allocation matters.
- You can adopt Kotlin: Arrow provides a more actively developed functional stack with language support for null-safety and data classes.
- You want minimal conceptual surface area for a team unfamiliar with functional idioms — Vavr is a large vocabulary to learn.

## Alternatives

- arrow-kt/arrow — richer functional library for Kotlin; use instead if you can adopt Kotlin and want first-class language null-safety.
- eclipse/eclipse-collections — high-performance mutable and immutable collections (including primitive specializations); use when collection performance and memory matter more than FP control types.
- functionaljava/functionaljava — older, purer FP library for Java; use if you want a more Haskell-flavored API and don't need Vavr's collection breadth.
- pcollections/pcollections — persistent collections only, no control types; use when you just need immutable data structures without the rest.
- jOOQ/jOOL — lightweight `Seq`/tuple/function extensions over `java.util.stream`; use when you want small functional sugar on top of standard streams rather than a parallel universe.

## History

| Version | Date | Notes |
|---------|------|-------|
| Javaslang 1.0 | 2014 | Initial release under the Javaslang name[^2]. |
| Javaslang 2.0 | 2016 | Major revision of the collection and control API[^2]. |
| Vavr 0.9.0 | 2017 | Renamed Javaslang → Vavr; `javaslang.*` → `io.vavr.*`[^2]. |
| Vavr 0.10.0 | 2019 | Long-lived stable line; the de facto production version for years. |
| Vavr 1.0.0 | — | Long-awaited stable major release after an extended alpha[^1]. |

## References

[^1]: vavr-io/vavr README and repository — description, maintainer, dependency-free packaging, current release. https://github.com/vavr-io/vavr
[^2]: Vavr documentation / project history on the Javaslang → Vavr rename and package migration. https://docs.vavr.io/

## Tags

java, functional-programming, immutable-collections, persistent-collections, object-functional, monad, pattern-matching, jvm, option-type, error-handling, library
