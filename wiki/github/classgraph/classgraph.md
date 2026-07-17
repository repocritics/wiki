# classgraph/classgraph

> A JVM classpath and module-path scanner that inverts the reflection API — find every class that extends, implements, or is annotated with X, without loading any of them.

[GitHub repo](https://github.com/classgraph/classgraph) ·
[Wiki / docs](https://github.com/classgraph/classgraph/wiki) ·
[License: MIT](https://github.com/classgraph/classgraph/blob/latest/LICENSE)

## Overview

ClassGraph is a parallelized classpath and module-path scanner for Java, Scala, Kotlin, and other JVM languages, written by Luke Hutchison (@lukehutch)[^1]. It reads classfile bytecode directly rather than loading classes, and builds an in-memory graph of every class, interface, annotation, method, and field visible to the JVM. The reflection API answers questions about one known class ("what does this class extend?"); ClassGraph answers the inverse across the whole classpath ("which classes extend this? which implement this interface? which carry this annotation?"), plus resource enumeration ("give me every file matching this path pattern in every classloader"). This inversion is what annotation-driven frameworks, plugin systems, and SPI discovery actually need.

The project was called **FastClasspathScanner** until the version 4 rewrite (2018), when it was renamed and the API was reworked around the `ClassGraph` builder and `ScanResult` model[^2]. It won a Duke's Choice Award at Oracle Code One 2018 and a Google Open Source Peer Bonus in 2022[^1]. It has no runtime dependencies and is compiled in Java 7 compatibility mode while still supporting the JPMS module system (JDK 9+) via reflection.

The defining tension is that **there is no standard way to enumerate a JVM classpath**. Fat jars, nested jars, Spring Boot's loader, OSGi, WAR/EAR containers, the JPMS module path, and Android's DEX format each expose classpath elements differently, and many classloaders only reveal their search path through private fields. ClassGraph's value is handling more of these mechanisms than any competitor — but the same reflective introspection it depends on is exactly what recent JDKs are locking down (see Production Notes).

## Getting Started

Maven (`io.github.classgraph:classgraph`, replace `X.Y.Z` with the latest release)[^3]:

```xml
<dependency>
    <groupId>io.github.classgraph</groupId>
    <artifactId>classgraph</artifactId>
    <version>X.Y.Z</version>
</dependency>
```

Find all classes annotated with `@com.xyz.Route`, without loading or initializing them:

```java
try (ScanResult scanResult = new ClassGraph()
        .enableAllInfo()             // scan classes, methods, fields, annotations
        .acceptPackages("com.xyz")   // limit scan scope (omit to scan everything)
        .scan()) {
    for (ClassInfo ci : scanResult.getClassesWithAnnotation("com.xyz.Route")) {
        AnnotationInfo ai = ci.getAnnotationInfo("com.xyz.Route");
        String route = (String) ai.getParameterValues().get(0).getValue();
        System.out.println(ci.getName() + " -> " + route);
    }
}
```

`ScanResult` is `AutoCloseable` and holds open resources — always use try-with-resources.

## Architecture / How It Works

ClassGraph works in two phases: a `ClassGraph` builder configures the scan, `.scan()` walks the classpath and returns a `ScanResult` holding the graph, and all queries run against that result object. Nothing is scanned lazily — the scan is eager and the result is a materialized model.

- **Direct bytecode parsing.** ClassGraph parses the classfile format itself, so it never invokes a classloader or runs static initializers. This avoids the side effects and cost of loading classes just to inspect them, and lets it read classes that could not even be loaded (missing transitive deps, wrong JDK target). `ClassInfo.loadClass()` is available but opt-in, and that is the only path that triggers real classloading.
- **Multithreaded, I/O-bound.** Scanning is parallelized to run close to disk/IO bandwidth limits; on an SSD the bottleneck is reading jar/class bytes, not CPU[^4].
- **Cost scales with what you enable.** `enableAllInfo()` reads method, field, and annotation detail for every class and is the expensive setting. Targeted toggles (`enableClassInfo`, `enableAnnotationInfo`, etc.) plus `acceptPackages` / `rejectPackages` scoping are how you keep scans cheap.
- **Classpath specification handling** is the hard, unglamorous core: it resolves Spring Boot fat jars, nested jars, OSGi/JBoss/WildFly loaders, servlet containers, the JPMS module path, and Android, normalizing them into one enumerable set.
- **Build-time scanning.** A `ScanResult` can be serialized to JSON and deserialized later, so scanning can happen at build time (e.g. Android annotation processing) and the graph reused at runtime without re-scanning.
- **Extras.** It can find duplicate class definitions on the classpath and emit GraphViz `.dot` visualizations of the class graph.

## Production Notes

**JDK 16+ strong encapsulation is the central operational risk.** Since JDK 16 enforced strong encapsulation (JEP 396), a classloader that only exposes its search path through private fields cannot be introspected by default. If your ClassGraph code works on JDK 11 but returns nothing on JDK 16+, this is almost always why[^5]. The escape hatches are two optional libraries — **Narcissus** (native code, limited to specific OS/arch combinations) or **JVM-Driver** (pure Java, roughly JDK 8–18) — set via `ClassGraph.CIRCUMVENT_ENCAPSULATION`. Both deliberately bypass access checks, and the maintainer's own guidance is that future JDKs may close these holes entirely, so the durable fix is to get your runtime's classloader to expose the classpath through a public API. Treat runtime classpath scanning on evolving JDKs as a standing portability liability.

**Always close `ScanResult`.** It retains open file handles and the in-memory graph; leaking it leaks both. Use try-with-resources.

**Memory.** The full graph lives in heap. On very large classpaths with `enableAllInfo()`, this is measurable — scope the scan or enable only what you query.

**Startup cost, not steady-state.** Scanning is a one-time startup expense. It is not something to run on a hot path; frameworks typically scan once at boot (or at build time) and cache the result.

**GraalVM native image.** Runtime classpath scanning fits poorly with native-image's closed-world assumption — the classpath as ClassGraph understands it does not exist the same way in a native binary. Build-time scanning with a serialized `ScanResult` is the more compatible pattern; expect friction otherwise.

**Version pinning.** The project is mature with a low bug rate, but ships frequent point releases in the 4.8.x line. The README explicitly warns against the `LATEST` Maven version for reproducible builds — pin an exact version[^3].

**Reflective-access warnings.** On some JDK/classloader combinations you may see illegal-reflective-access warnings on the console; switching the reflection driver to JVM-Driver is one way to quiet them.

## When to Use / When Not

**Use when:**
- You need annotation-driven discovery: find every `@Entity`, `@Route`, `@Plugin`, or SPI implementation across the classpath.
- You want to enumerate implementations/subclasses without loading (and thereby initializing) them.
- You need resource discovery — all files matching a path or extension across every classloader/module.
- You want build-time annotation processing (Android) or a serialized class-graph reused at runtime.
- You run in an awkward container (Spring Boot fat jar, OSGi, WAR) where naive classpath enumeration fails.

**Avoid when:**
- You already know the exact class names — plain reflection or direct references are simpler and free.
- You only need one class's metadata — the standard reflection API is the right tool.
- You are on GraalVM native image and cannot move scanning to build time.
- Your JDK/classloader environment locks down reflection and you cannot ship Narcissus/JVM-Driver — factor in that portability risk before depending on runtime scanning.

## Alternatives

- ronmamo/reflections — the most common alternative; simpler API, widely used, but covers fewer real-world classpath mechanisms and is less rigorous about JPMS/container edge cases.
- wildfly/jandex — builds an offline annotation index at build time (used by Quarkus, Hibernate); extremely fast lookups but requires a pre-generated index rather than live scanning.
- atteo/classindex — compile-time annotation processor that writes an index at build time; use when you want zero runtime scan cost and control the source.
- burningwave/core — broader JVM-internals toolkit by the JVM-Driver author; use when you need reflection/driver capabilities beyond scanning.
- Spring's built-in classpath scanning — use when you are already in a Spring context and don't want a separate dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2014-08 | Repo created as **FastClasspathScanner**[^1]. |
| 4.0 | 2018 | Renamed to ClassGraph; API rewritten around `ClassGraph` builder + `ScanResult`[^2]. |
| — | 2018 | Duke's Choice Award, Oracle Code One[^1]. |
| — | 2022 | Google Open Source Peer Bonus[^1]. |
| 4.8.x | 2026 | Ongoing point releases on the 4.8 line; last pushed 2026-06-05[^3]. |

## References

[^1]: ClassGraph README and repository. https://github.com/classgraph/classgraph
[^2]: Porting FastClasspathScanner code to ClassGraph (wiki). https://github.com/classgraph/classgraph/wiki/Porting-FastClasspathScanner-code-to-ClassGraph
[^3]: Maven Central artifact `io.github.classgraph:classgraph`. https://mvnrepository.com/artifact/io.github.classgraph/classgraph
[^4]: "How fast is ClassGraph?" (wiki). https://github.com/classgraph/classgraph/wiki/How-fast-is-ClassGraph
[^5]: ClassGraph README, "Running on JDK 16+" — strong encapsulation and the Narcissus / JVM-Driver reflection drivers. https://github.com/classgraph/classgraph#running-on-jdk-16

## Tags

java, jvm, classpath-scanner, annotation-scanning, reflection, bytecode, module-path, jpms, scala, kotlin, metaprogramming
