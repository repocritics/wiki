# ronmamo/reflections

> Runtime classpath metadata scanning for Java — reverse-query the type system for subtypes, annotated elements, and members.

[GitHub repo](https://github.com/ronmamo/reflections) ·
[License: WTFPL](https://github.com/ronmamo/reflections/blob/master/COPYING.txt)

## Overview

Reflections scans a project's classpath at startup, reads class metadata straight from bytecode, and builds an in-memory index you can query in reverse: "which classes extend `X`", "which types carry `@Y`", "which methods return `void`", "which resources match a regex"[^1]. The standard Java reflection API only answers forward questions about a class you already hold; Reflections inverts that, giving you the transitive closure of the type system without you enumerating packages by hand.

It became a near-ubiquitous transitive dependency in the Java ecosystem — pulled in by plugin loaders, DI wiring, ORM entity discovery, test harnesses, and serializers that need to find implementations by convention rather than explicit registration. The library was written "in the spirit of Scannotations"[^1] and leans on Javassist[^2] to inspect classes without loading them, which keeps scanning cheap and side-effect-free relative to `Class.forName`.

The defining tension is maintenance versus reach. The README states plainly that the project is **not under active development or maintenance**, with the last release `0.10.2` in October 2021[^1] and 100+ open issues left to workarounds. Yet it still sits behind a large slice of the JVM world. That gap — heavy real-world usage against a frozen codebase — is the single most important thing to know before adopting it in 2026.

## Getting Started

```xml
<!-- Maven -->
<dependency>
    <groupId>org.reflections</groupId>
    <artifactId>reflections</artifactId>
    <version>0.10.2</version>
</dependency>
```

```gradle
// Gradle
implementation 'org.reflections:reflections:0.10.2'
```

```java
import org.reflections.Reflections;
import org.reflections.util.ConfigurationBuilder;
import org.reflections.util.FilterBuilder;
import static org.reflections.scanners.Scanners.*;

Reflections reflections = new Reflections(
  new ConfigurationBuilder()
    .forPackage("com.my.project")
    .filterInputsBy(new FilterBuilder().includePackage("com.my.project"))
    .setScanners(SubTypes, TypesAnnotated));

// classes that implement/extend SomeType
Set<Class<?>> subTypes =
  reflections.get(SubTypes.of(SomeType.class).asClass());

// classes annotated with @SomeAnnotation
Set<Class<?>> annotated =
  reflections.get(SubTypes.of(TypesAnnotated.with(SomeAnnotation.class)).asClass());
```

## Architecture / How It Works

Scanning happens once, up front. `ConfigurationBuilder` resolves a set of classpath URLs (from a package name, classloader, or explicit URLs), then each configured `Scanner` walks those URLs. Rather than loading classes into the JVM, Reflections parses `.class` files with Javassist[^2] and records only the metadata it cares about. Results accumulate in a `Store` — effectively a multimap keyed by scanner, where values are string keys (class names, annotation names) mapped to sets of matching element names.

`Scanners` is the query entry point. Standard scanners include `SubTypes`, `TypesAnnotated`, `MethodsAnnotated`, `FieldsAnnotated`, `MethodsReturn`, `MethodsSignature`, `MethodsParameter`, `ConstructorsAnnotated`, and `Resources`. Each implements a fluent `QueryBuilder` / `QueryFunction` interface supporting `get()` (direct values) versus `of()`/`with()` (transitive closure), plus `filter()`, `map()`, `flatMap()`, and `as()`/`asClass()` for composition. The `0.10` line also exposes `ReflectionUtils` (`SuperTypes`, `Fields`, `Methods`, `Annotations`, …) that operates on live `Class` objects via standard reflection, so you can chain scanned metadata into reflective queries in one expression.

Because it works name-first, class objects are only materialized when you call `.asClass()` — resolution goes through the configured `ClassLoader`, which is where most runtime surprises originate. A scanned name that cannot be resolved (missing dependency, wrong loader) throws at query time, not scan time.

Two persistence paths exist for skipping scan cost: `Reflections.save()` serializes the `Store` to XML/JSON at build time, collected later with `Reflections.collect()`[^1]; and `JavaCodeSerializer` emits scanned metadata as generated Java source for strongly-typed access. The `MemberUsageScanner` (experimental) records member-to-member usages for cross-package/layer analysis.

## Production Notes

**Silent empty results are the classic footgun.** A scanner must be explicitly configured to be queried — query an unconfigured scanner and you get an empty set, not an error. The default scanners are only `SubTypes` and `TypesAnnotated`; everything else (methods, fields, resources) requires `.setScanners(...)`. Teams routinely lose hours to "the query returns nothing" that is really a missing scanner.[^1]

**Annotation retention.** `TypesAnnotated`/`MethodsAnnotated` only see annotations with `RetentionPolicy.RUNTIME` (or `CLASS`, since bytecode is read directly). `SOURCE`-retained annotations are invisible.

**Classpath resolution is the hard part at scale.** `forPackage`/URL discovery works cleanly on flat classpaths but is fragile with non-standard packaging: Spring Boot fat jars (nested `BOOT-INF/classes`), shaded/uber jars, custom classloaders, and JPMS modules (Java 9+) all change how URLs enumerate. Under-filtered configuration scans the entire classpath including third-party jars, which is slow and noisy; the README repeatedly urges `filterInputsBy()` to constrain inputs. Over-filtered configuration silently drops classes you needed.

**The 0.9 → 0.10 break.** `0.10` rewrote the API around `Scanners`/`QueryFunction` and removed the old scanner classes (`SubTypesScanner`, `TypeAnnotationsScanner`, etc.). The legacy `getSubTypesOf` / `getTypesAnnotatedWith` calls still work as a compatibility layer, but a lot of internet copy-paste predates the change and won't compile against `0.10`. Pin your version and match tutorials to it.

**Maintenance risk.** No release since October 2021 and an explicit "not maintained" banner mean bugs on newer JDKs, packaging formats, or transitive Javassist CVEs will not be fixed upstream. It works well on the JDK/packaging shapes that existed when it was frozen; treat anything newer as your problem to validate. For new projects this is the strongest reason to evaluate `classgraph` first.

**Startup cost.** Scanning is proportional to the classpath surface you don't filter out. In serverless/short-lived processes the scan tax hits every cold start — the build-time `save()`/`collect()` path exists precisely to move that cost out of runtime.

## When to Use / When Not

**Use when:**
- You have an existing codebase already depending on it and it works — there's no urgency to migrate a stable integration.
- You need convention-based discovery (plugins, handlers, entities) and want a small, familiar API.
- You can constrain scanning to your own packages and control the classloader.

**Avoid when:**
- You're starting fresh in 2026 — prefer a maintained scanner (classgraph).
- You run on non-trivial packaging (Spring Boot fat jars, JPMS, GraalVM native image) where an unmaintained scanner's classpath assumptions may not hold. GraalVM native image in particular disallows arbitrary runtime reflection without registration, undercutting the whole approach.
- Cold-start latency matters and you can't move scanning to build time.

## Alternatives

- classgraph/classgraph — actively maintained, faster, broader classpath/module support; the default recommendation for new work.
- google/guava — `ClassPath` does lightweight top-level class enumeration only; no annotation/subtype indexing.
- spring-projects/spring-framework — `ClassPathScanningCandidateComponentProvider` covers annotation/assignable scanning if you're already in Spring.
- atteo/classindex — compile-time annotation indexing via an annotation processor; zero runtime scan cost, but only what you annotate.
- Java's built-in `ServiceLoader` (`META-INF/services`) — explicit registration instead of scanning; native-image friendly.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2013-04 | Repository created; classpath scanning around Javassist[^2]. |
| 0.9.x | 2014–2019 | Long-lived line: `SubTypesScanner`, `TypeAnnotationsScanner`, `getSubTypesOf` style API. |
| 0.10.0 | 2021 | Rewrite: functional `Scanners`/`QueryFunction` API, `ReflectionUtils` query builders, old scanners removed. |
| 0.10.2 | 2021-10 | Last released version[^1]. |
| — | 2024-06 | Last commit to `master`; no release. Marked not maintained[^1]. |

## References

[^1]: Reflections README and usage guide. https://github.com/ronmamo/reflections
[^2]: Javassist — bytecode manipulation library used for reading class metadata without loading. https://github.com/jboss-javassist/javassist

## Tags

java, jvm, reflection, classpath-scanning, annotations, metadata, bytecode, dependency-injection, unmaintained, javassist
