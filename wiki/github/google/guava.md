# google/guava

> Google's core Java libraries — collections, caching, hashing, concurrency, and string/IO utilities extracted from Google's internal codebase.

[GitHub repo](https://github.com/google/guava) ·
[Official website](https://guava.dev/) ·
[License: Apache-2.0](https://github.com/google/guava/blob/master/COPYING)

## Overview

Guava is the set of general-purpose Java utilities that Google open-sourced starting in 2007 (as "Google Collections") and consolidated under the Guava name around 2010[^1]. It fills the gap between the JDK standard library and application code: immutable collections, new collection types (`Multimap`, `Multiset`, `BiMap`, `Table`, `RangeSet`), an in-memory caching library, a functional-hashing API, a graph library, and hardened helpers for strings, primitives, I/O, and concurrency (`ListenableFuture`). It is one of the most-depended-upon artifacts in the entire JVM ecosystem, transitively present in a large fraction of Java projects.

Guava's defining tension is that much of what it originally provided has since landed in the JDK itself. Java 8 (2014) introduced streams, `Optional`, and lambdas, superseding Guava's `Function`/`Predicate`/`FluentIterable` and much of its functional idiom; `java.time` replaced parts of its date handling. What remains uniquely valuable — immutable collections with a clean builder API, `Cache`/`LoadingCache`, `Multimap`/`Table`, consistent hashing, `Preconditions`, and `ListenableFuture` — is enough to keep it ubiquitous, but new code on modern JDKs should reach for standard-library equivalents first and Guava second.

The second tension is dependency weight and churn. Guava is a large jar (~3 MB) pulled in by countless libraries, which makes version conflicts (diamond dependencies) a recurring build-time headache, and its historical willingness to remove `@Beta` APIs has burned downstream libraries.

## Getting Started

Guava's Maven group ID is `com.google.guava` and artifact ID is `guava`. It ships in two flavors distinguished by version suffix: `-jre` (Java 8+) and `-android` (backported for Android and older bytecode)[^2].

```xml
<dependency>
  <groupId>com.google.guava</groupId>
  <artifactId>guava</artifactId>
  <version>33.6.0-jre</version>
</dependency>
```

```java
import com.google.common.collect.*;
import com.google.common.cache.*;
import java.util.concurrent.TimeUnit;

// Immutable collections with a builder
ImmutableList<String> names = ImmutableList.of("a", "b", "c");

// Multimap: one key -> many values
Multimap<String, Integer> m = ArrayListMultimap.create();
m.put("even", 2); m.put("even", 4);   // {even=[2, 4]}

// A loading cache with size + time eviction
LoadingCache<String, Integer> cache = CacheBuilder.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(10, TimeUnit.MINUTES)
    .build(CacheLoader.from(String::length));
int len = cache.getUnchecked("hello");  // 5
```

## Architecture / How It Works

Guava is a single flat jar organized into `com.google.common.*` packages, each a largely independent utility area: `collect`, `cache`, `hash`, `graph`, `io`, `net`, `primitives`, `math`, `base`, and `util.concurrent`. There is no runtime container or framework — everything is static helpers and concrete classes.

- **Collections** are the historical core. Immutable variants (`ImmutableList`, `ImmutableMap`, etc.) are genuinely immutable, null-hostile, and often more memory-compact than their JDK counterparts. `Multimap`, `Multiset`, `BiMap`, `Table`, and `RangeMap`/`RangeSet` provide data shapes the JDK never added.
- **Cache** (`CacheBuilder`/`LoadingCache`) is an in-process, concurrent, segmented cache with size- and time-based eviction, weak/soft references, and removal listeners. It is the direct ancestor of Caffeine, which the same author (Ben Manes) later wrote as a faster replacement[^3].
- **`ListenableFuture`** predates `CompletableFuture` and remains the interop point for a lot of async Java (notably gRPC). `Futures` provides the combinators.
- **`hash`** offers non-cryptographic and cryptographic `HashFunction`s (murmur3, sipHash, CRC, SHA family) plus `BloomFilter`.
- **`base`** holds `Preconditions`, `Optional` (Guava's, distinct from `java.util.Optional`), `Splitter`/`Joiner`, `Strings`, and `Stopwatch`.

The dual-flavor build is a real architectural constraint: the `-jre` and `-android` sources diverge (the Android flavor avoids Java 8 APIs and streams), and a single `@Beta`/`@GwtCompatible`/`@GwtIncompatible` annotation matrix governs which APIs are portable to GWT and Android. Guava has exactly one runtime dependency, `com.google.guava:failureaccess`, plus annotation-only compile dependencies (JSR-305, Checker, Error Prone)[^4].

## Production Notes

**Version conflicts are the #1 operational cost.** Because Guava is transitively depended on everywhere, large builds routinely resolve conflicting versions. Guava keeps non-`@Beta` APIs binary-compatible indefinitely — the last release to remove non-`@Beta` APIs was 21.0[^5] — so pinning to the newest resolved version via a BOM or `dependencyManagement` is the standard fix.

**`@Beta` is a real trap for library authors.** APIs annotated `@Beta` can change or be removed in any release. If you are writing a *library* (not an app), using a `@Beta` Guava API can break your consumers on a Guava upgrade. Google ships the [Guava Beta Checker] Error Prone plugin specifically to fail the build if you touch `@Beta` from library code.

**Prefer Caffeine over Guava cache for new code.** Guava's `Cache` still works, but Caffeine (`com.github.ben-manes.caffeine`) is faster, has a near-identical API, and is where the author's effort went. Guava cache remains fine for modest, low-contention caches and for avoiding an extra dependency.

**Serialized forms are not stable.** Guava explicitly states serialized forms of its objects may change between versions — never persist a serialized Guava object and expect a future version to read it.

**Not a security boundary.** Guava's classes are not designed to defend against malicious callers; do not use them at trust boundaries between trusted and untrusted code.

**Platform caveats.** Some `com.google.common.io` behavior is only tested on Linux/Windows and may differ elsewhere. The `-jre` flavor requires JDK 8+; use the `-android` flavor for Android or older bytecode targets. Guava is heavily null-hostile — passing `null` into collections or utilities that reject it throws `NullPointerException` by design.

## When to Use / When Not

**Use when:**
- You need collection types the JDK lacks: `Multimap`, `Multiset`, `BiMap`, `Table`, `RangeSet`.
- You want clean immutable collections with builders and defensive copies.
- You need in-process caching, consistent/Bloom hashing, or `ListenableFuture` interop (e.g. gRPC).
- The dependency is already on your classpath (it usually is) and you want battle-tested helpers.

**Avoid / reconsider when:**
- You're on a modern JDK and the JDK already covers it: streams, `java.util.Optional`, `Map.of`/`List.of`, `record`s, and `java.time` replace much of early Guava.
- You need the fastest possible in-memory cache — use Caffeine.
- You're minimizing jar size / cold-start footprint in a small service or serverless function.
- You're a library author tempted by `@Beta` APIs — the future breakage isn't worth it.

## Alternatives

- ben-manes/caffeine — use instead of `guava-cache` when you need a faster, actively-optimized in-process cache with a near-identical API.
- apache/commons-lang — use instead when you want string/reflection/utility helpers under the Apache Commons umbrella rather than Guava's collection-first design.
- eclipse/eclipse-collections — use instead when you want richer, higher-performance primitive and immutable collections as your project's collection backbone.
- vavr-io/vavr — use instead when you want functional-programming persistent data structures and `Try`/`Either`/`Option` idioms.
- Plain JDK (Java 17/21) — use instead when standard-library `List.of`, `Optional`, streams, and `java.time` already meet the need without adding a dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| Google Collections 1.0 | 2010-01 | Precursor: immutable collections, `Multimap`, `Multiset`[^1]. |
| Guava r05–r09 | 2010–2011 | Rebranded and expanded beyond collections (cache, hash, base). |
| 10.0 | 2011-09 | First semver-style release; `CacheBuilder` introduced. |
| 21.0 | 2016-12 | Last release to remove non-`@Beta` APIs; baseline for JDK 8[^5]. |
| 30.0 | 2020-10 | Split out `failureaccess`; dependency and annotation cleanup. |
| 33.6.0 | 2026 | Current line; `-jre` (JDK 8+) and `-android` flavors[^2]. |

## References

[^1]: Kevin Bourrillion et al., Google Collections Library → Guava. https://github.com/google/guava/wiki/Release10
[^2]: Guava README, "Adding Guava to your build" — JRE vs Android flavors. https://github.com/google/guava/blob/master/README.md
[^3]: Caffeine caching library by Ben Manes, successor to Guava's cache. https://github.com/ben-manes/caffeine
[^4]: Guava dependencies discussion (failureaccess + annotation-only deps). https://github.com/google/guava/wiki/UseGuavaInYourBuild
[^5]: Guava README, "IMPORTANT WARNINGS" — binary compatibility and last non-`@Beta` removal in 21.0. https://github.com/google/guava/blob/master/README.md

[Guava Beta Checker]: https://github.com/google/guava-beta-checker

## Tags

java, jvm, collections, caching, hashing, concurrency, immutable-collections, utility-library, google, maven, standard-library
