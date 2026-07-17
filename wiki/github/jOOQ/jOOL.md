# jOOQ/jOOL

> Tuples, checked-exception wrappers, and a richer sequential Stream API for Java 8+ — "the missing parts" the JDK's lambda work left out.

[GitHub repo](https://github.com/jOOQ/jOOL) ·
[Official website](http://www.jooq.org/products) ·
[License: Apache-2.0](https://github.com/jOOQ/jOOL/blob/main/LICENSE)

## Overview

jOOλ (spelled "jOOL" in package and artifact names) is a small utility
library from Data Geekery, the company behind jOOQ, that fills gaps in Java 8's
functional additions[^1]. It bundles three largely independent things:
type-safe tuples (`Tuple1` through `Tuple16`), extended function interfaces
(`Function1` through `Function16` and checked variants), and `Seq`, an
interface that extends `java.util.stream.Stream` with dozens of operations
borrowed from Scala and other functional languages — `zipWithIndex`,
`foldLeft`/`foldRight`, `groupBy` (returning a map, not a collector), window
functions, joins, `duplicate`, `crossJoin`, and so on.

The library's defining premise is that the JDK 8 Expert Group prioritized
backwards compatibility and parallelism, and in doing so shipped a Stream API
that is awkward for the common single-threaded, ordered case. jOOλ's answer is
deliberately narrow: every `Seq` is **sequential and ordered** by design —
calling `parallel()` on one returns the same sequential `Seq`[^2]. That is the
central tradeoff. You trade the JDK's parallelism story for ergonomics on the
80% case, plus tuples and a genuinely useful escape hatch for checked
exceptions inside lambdas.

It is a mature, low-churn library rather than an actively evolving one. The
current release is 0.9.x and it has never cut a 1.0; the repository's last
substantive push was in 2024[^3]. For a dependency-free helper library that is
closer to "finished" than "abandoned," but new features are rare.

## Getting Started

Maven, targeting Java 9 or later:

```xml
<dependency>
  <groupId>org.jooq</groupId>
  <artifactId>jool</artifactId>
  <version>0.9.15</version>
</dependency>
```

For Java 8, use the `jool-java-8` artifact at the same version[^4].

```java
import org.jooq.lambda.Seq;
import static org.jooq.lambda.tuple.Tuple.tuple;

// zipWithIndex + a real groupBy returning a Map
Seq.of("a", "b", "c").zipWithIndex();
// (("a",0), ("b",1), ("c",2))

Seq.of(1, 2, 3, 4).groupBy(i -> i % 2);
// { 1=[1, 3], 0=[2, 4] }   -- a java.util.Map, not a downstream Stream

// Wrap a checked exception so it fits a JDK functional interface
import org.jooq.lambda.Unchecked;
Arrays.stream(dir.listFiles())
      .map(Unchecked.function(File::getCanonicalPath))  // throws IOException
      .forEach(System.out::println);
```

## Architecture / How It Works

`Seq<T> extends Stream<T>`, so a `Seq` is a decorator: it holds a delegate
`Stream` and implements the extra methods on top of the inherited terminal and
intermediate operations. This is why the sequential-only constraint exists —
many of the added operations (`zipWithIndex`, `foldLeft`, window functions,
`duplicate`) have no meaningful or efficient parallel semantics on an ordered
stream, so the library refuses to pretend otherwise. Most `Seq` operations are
still lazy where the underlying `Stream` is lazy, but several (`duplicate`,
`reverse`, `shuffle`, `sorted`-style window setup) are inherently buffering and
will materialize the stream.

The tuple types are plain generated classes: `Tuple2<T1,T2>` with public final
`v1`/`v2` fields, `Comparable`, `Serializable`, and helpers like `map1`,
`concat`, and `swap`. Because there are exactly 16 arities, everything is
statically typed with no varargs erasure — the cost is 16 near-identical
generated classes per family (tuples, functions, consumers, checked variants).

`Unchecked` (with siblings `Sneaky` and `Try`) works by the standard trick of
catching a checked exception in a wrapper lambda and rethrowing it through
`sneakyThrow`, exploiting generic type erasure so the compiler never sees the
checked type. `Unchecked.throwChecked(new Exception())` will throw a checked
exception from a context the compiler believes cannot. This is convenient and
also exactly the kind of thing that surprises the next reader of the stack
trace.

There are no runtime dependencies. The whole library is a single jar of
generated and hand-written helpers against `java.util.stream`.

## Production Notes

- **Sequential by design.** `Seq` will not parallelize. If a hot path needs
  fork-join parallelism, jOOλ is the wrong tool — use raw `Stream.parallel()`
  and give up the sugar. Do not expect `.parallel()` on a `Seq` to do anything.
- **Buffering operations cost memory.** `duplicate()`, `reverse()`,
  `zip`, and the join/`crossJoin` family have to hold elements. On large or
  unbounded streams these will either OOM or never terminate. `cycle()` is
  infinite by definition. Treat `Seq` as a collections convenience, not a
  streaming-data-pipeline engine.
- **`Unchecked` hides checked exceptions from the compiler, not from
  runtime.** Sneaky-throw means callers cannot `catch (IOException e)` around
  the lambda in the usual way and the compiler won't warn them to. It is a
  sharp tool; document its use at call sites.
- **`groupBy` semantics differ from `Collectors.groupingBy`.** jOOλ's
  `Seq.groupBy` eagerly returns a `Map` (a terminal operation); the JDK
  collector composes downstream collectors. They are not drop-in equivalents.
- **Two artifacts, one project.** `jool` targets Java 9+ (real module-friendly
  build), `jool-java-8` targets Java 8. Picking the wrong one on an older JVM
  produces class-version errors at load time. Keep versions aligned.
- **Maintenance cadence is slow.** With no 1.0 and infrequent releases[^3],
  don't adopt it expecting rapid fixes or new JDK-feature coverage. For stable
  helpers this is fine; for anything you'd need patched quickly, weigh the
  risk. It has no relationship to jOOQ's commercial support tiers.

## When to Use / When Not

**Use when:**
- You want tuples in Java without hand-rolling them or pulling in a larger
  framework.
- You do a lot of ordered, single-threaded stream work and miss `zipWithIndex`,
  `foldLeft`, `window`, or a `Map`-returning `groupBy`.
- Checked exceptions inside lambdas are a recurring friction point and you want
  `Unchecked`/`Try` wrappers.
- You want zero transitive dependencies.

**Avoid when:**
- Your workload needs parallel streams — `Seq` refuses to parallelize.
- You're on a modern Java where records (Java 16+) already give you tuple-like
  value types, reducing the tuple case.
- You want an actively evolving functional library; jOOλ is effectively in
  maintenance mode.
- You're processing large or infinite data where the buffering operations are
  landmines.

## Alternatives

- javaslang/vavr — broader functional library (persistent collections, `Try`,
  `Either`, `Option`, tuples); use when you want a whole functional ecosystem,
  not just Stream sugar.
- jOOQ/jOOR — sibling library for fluent reflection; use for the reflection
  problem, not stream/tuple ergonomics.
- google/guava — `FluentIterable`, `Table`, immutable collections; use when you
  want mature, widely-vetted collection utilities over stream extensions.
- amaembo/streamex — the closest competitor: an extended `Stream` API that
  *does* keep parallelism; use when you want extra operators without giving up
  parallel streams.
- projectreactor/reactor-core — reactive streams; use when the real need is
  async/backpressured data flow rather than eager collections work.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2014-03 | Repository created; tuples + `Seq` + `Unchecked`[^5]. |
| 0.9.x | 2015 onward | Long-lived 0.9 line; incremental additions to `Seq`. |
| jool-java-8 split | — | Separate artifact introduced for Java 8 vs 9+ builds[^4]. |
| 0.9.15 | current | Latest published release; last major push 2024[^3]. |

## References

[^1]: jOOλ README — "The Missing Parts in Java 8." https://github.com/jOOQ/jOOL
[^2]: jOOλ README, `org.jooq.lambda.Seq` — "all Seq's are sequential and ordered streams." https://github.com/jOOQ/jOOL
[^3]: jOOQ/jOOL repository metadata (GitHub API): 0.9.15 latest, last push 2024-08-01, no 1.0 release. https://github.com/jOOQ/jOOL
[^4]: jOOλ README, Download section — `jool` (Java 9+) vs `jool-java-8` artifacts, version 0.9.15. https://github.com/jOOQ/jOOL
[^5]: jOOQ/jOOL repository, created 2014-03-02 (GitHub API). https://github.com/jOOQ/jOOL

## Tags

java, java-8, functional-programming, streams, tuples, lambda, jvm, utility-library, jooq, sequential-streams
