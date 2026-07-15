# apache/commons-collections

> Extensions to the Java Collections Framework — Bag, BidiMap, MultiValuedMap, and a decorator toolkit — and the library at the center of the 2015 Java deserialization crisis.

[GitHub repo](https://github.com/apache/commons-collections) ·
[Official website](https://commons.apache.org/proper/commons-collections/) ·
[License: Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0)

## Overview

Apache Commons Collections is one of the oldest Java utility libraries still in active use, with a first release in 2001[^1] and a repository history on GitHub reaching back to 2009. It provides collection types and helpers that the JDK's `java.util` framework never shipped: `Bag` (a multiset), `BidiMap` (bidirectional maps), `MultiValuedMap` (keys mapping to multiple values), ordered maps and sets, plus a large family of decorators that wrap existing collections to add behavior — predicate-checked, type-transformed, unmodifiable, lazily-populated.

The library predates Java generics, streams, and much of what modern Java developers reach for by default. For a decade it was a near-universal dependency; today its role is narrower. Java 5 generics, Java 8 streams, `Map.of`/`List.of` factories, and third-party libraries (notably Guava) have absorbed much of what once made Commons Collections essential. What remains distinctive are the data structures the JDK still lacks — bags, bidi-maps, multi-valued maps — and the decorator pattern for composing collection behavior.

The project carries a defining historical burden: its `InvokerTransformer` and related "functor" classes formed the most famous Java deserialization gadget chain, used to achieve remote code execution against any application that deserialized untrusted data with Commons Collections on the classpath[^2]. This was not a bug in the library's intended behavior so much as a demonstration that Java's native serialization is dangerous — but Commons Collections became the canonical example, and shaped how the ecosystem now treats deserialization.

## Getting Started

Maven (the current major line is `commons-collections4`, a distinct artifact and Java package from the legacy `commons-collections` 3.x):

```xml
<dependency>
  <groupId>org.apache.commons</groupId>
  <artifactId>commons-collections4</artifactId>
  <version>4.5.0</version>
</dependency>
```

```java
import org.apache.commons.collections4.Bag;
import org.apache.commons.collections4.bag.HashBag;
import org.apache.commons.collections4.BidiMap;
import org.apache.commons.collections4.bidimap.DualHashBidiMap;

Bag<String> bag = new HashBag<>();
bag.add("apple", 3);              // add with a count
System.out.println(bag.getCount("apple"));   // 3

BidiMap<String, Integer> ids = new DualHashBidiMap<>();
ids.put("alice", 1);
System.out.println(ids.getKey(1));           // "alice" — reverse lookup
```

## Architecture / How It Works

The library is organized around a small number of interfaces and one dominant design pattern.

**Core interfaces** extend or parallel `java.util` types: `Bag` / `SortedBag` (counted multisets), `BidiMap` / `SortedBidiMap` / `OrderedBidiMap`, `MultiValuedMap`, `MultiMap` (legacy), `OrderedMap`, `Trie`. Each has one or more concrete backing implementations (`HashBag`, `TreeBag`, `DualHashBidiMap`, `PatriciaTrie`, `ArrayListValuedHashMap`, and so on).

**Decorators** are the library's signature. Rather than subclassing, you wrap a collection to layer in a cross-cutting concern: `PredicatedList` rejects elements failing a predicate, `TransformedCollection` transforms elements on insertion, `UnmodifiableMap` blocks mutation, `LazyMap` computes absent values on read via a `Factory`. Decorators compose, so a collection can be predicated-then-unmodifiable. These build on the **functor** abstractions — `Predicate`, `Transformer`, `Closure`, `Factory` — small single-method interfaces that predate Java 8's functional interfaces and overlap with them conceptually.

**Static utility classes** are the most-used surface in practice: `CollectionUtils`, `ListUtils`, `SetUtils`, `MapUtils`, `IterableUtils`, `IteratorUtils`. These offer null-safe operations, set-theoretic helpers (`union`, `intersection`, `disjunction`), and emptiness checks that predate `Objects.requireNonNullElse` and stream collectors.

The `InvokerTransformer` functor — which invokes an arbitrary named method via reflection on whatever object flows through it — is what made the library weaponizable. Chained through `ChainedTransformer` and triggered by a `LazyMap` or `TransformedMap` during deserialization, it let an attacker run `Runtime.exec` from a crafted serialized byte stream. The library still contains these classes for backward compatibility, but since 3.2.2 / 4.1 they refuse to deserialize by default[^2].

## Production Notes

**The 3.x vs 4.x split is real and load-bearing.** `commons-collections` (groupId and artifactId both `commons-collections`, package `org.apache.commons.collections`) is the legacy 3.x line. `commons-collections4` (groupId `org.apache.commons`, package `org.apache.commons.collections4`) is the current line. They use different packages specifically so both can sit on one classpath during migration — a transitive dependency may drag in either. New code should target 4.x; audit for 3.2.1-or-earlier arriving transitively.

**Deserialization is the security story that will not die.** If you deserialize untrusted data with any version of Commons Collections reachable, you have a problem regardless of the library version — the correct fix is to not deserialize untrusted data, or to use a serialization filter (`java.io.ObjectInputFilter`, JEP 290, Java 9+). Commons Collections 3.2.2 and 4.1 disabled unsafe functor deserialization by default; a system property re-enables it, and you should never set it. Static analysis and dependency scanners still flag old versions aggressively.

**Much of it is superseded.** For null-safe checks and stream-based transformation, modern Java and Guava frequently do the job with no extra dependency reasoning. Reach for Commons Collections when you specifically need a `Bag`, a `BidiMap`, a `MultiValuedMap`, a `PatriciaTrie`, or the decorator composition model — not as a default utility grab-bag. Guava's `Multiset`, `BiMap`, and `Multimap` cover the same three core structures with a more modern API and are often the better new-code choice.

**Primitive collections are out of scope.** Commons Collections boxes everything. If memory or throughput on `int`/`long` collections matters, Eclipse Collections or fastutil are the right tools; Commons Collections will not help.

**Release cadence is slow and stable.** This is mature ASF infrastructure — expect infrequent releases, careful backward compatibility, and Java baseline moves that lag the ecosystem. Version 4.5.0 targets Java 8. Do not expect rapid feature development; expect longevity.

## When to Use / When Not

**Use when:**
- You need a multiset (`Bag`), bidirectional map (`BidiMap`), or multi-valued map (`MultiValuedMap`) and don't already have Guava.
- You want composable collection decorators (predicated, transformed, unmodifiable, lazy).
- You maintain a legacy codebase already built on its `CollectionUtils` / `MapUtils` helpers.
- You need a PATRICIA trie or specialized ordered-map structures the JDK lacks.

**Avoid when:**
- Plain JDK streams, `Map.of`/`List.of`, and `Objects` helpers already cover your needs.
- You're starting fresh and Guava is (or could be) on the classpath — its equivalents are more idiomatic.
- You need primitive-specialized collections (use Eclipse Collections or fastutil).
- You need persistent/immutable functional collections (use Vavr or Guava's immutable collections).

## Alternatives

- google/guava — `Multiset`, `BiMap`, `Multimap`, immutable collections; the default modern choice for the same three core structures.
- eclipse/eclipse-collections — memory-efficient collections including primitive specializations; richer functional API.
- vavr-io/vavr — persistent, immutable, functional-style collections for a more Scala-like Java.
- apache/commons-lang — sibling ASF library for `String`/`Object`/reflection utilities; complementary, not a replacement.
- The JDK itself — for many null-safe and transformation needs, `java.util.stream` plus `Map.of`/`List.of` removes the dependency entirely.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2001 | Initial release; pre-generics `java.util` extensions[^1]. |
| 3.0 | 2004 | Major rework of the collection type hierarchy. |
| 3.2.1 | 2008 | Long-lived 3.x baseline; the version most gadget chains targeted. |
| 4.0 | 2013 | New Maven coordinates + package (`commons-collections4`), generics throughout[^3]. |
| 3.2.2 | 2015-11 | Security hardening — unsafe functor deserialization disabled by default[^2]. |
| 4.1 | 2015-11 | Same deserialization hardening applied to the 4.x line[^2]. |
| 4.4 | 2019 | Java 8 baseline; API refinements. |
| 4.5.0 | 2024 | Current release; new utilities and fixes on the Java 8 baseline[^4]. |

## References

[^1]: Apache Commons Collections homepage and project history. https://commons.apache.org/proper/commons-collections/
[^2]: Apache Commons Collections security reports — deserialization of `InvokerTransformer` and related functors; mitigations in 3.2.2 and 4.1. https://commons.apache.org/proper/commons-collections/security-reports.html
[^3]: Apache Commons Collections 4.0 release notes — new package and Maven artifact `commons-collections4`. https://commons.apache.org/proper/commons-collections/release_4_0.html
[^4]: Apache Commons Collections release history / changes report. https://commons.apache.org/proper/commons-collections/changes-report.html

## Tags

java, collections, data-structures, jvm, apache-commons, multiset, bidimap, utility-library, deserialization-security, decorator-pattern
