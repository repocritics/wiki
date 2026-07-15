# apache/commons-lang

> Null-safe utility classes for `java.lang` — the string, number, and object helpers the JDK never shipped.

[GitHub repo](https://github.com/apache/commons-lang) ·
[Official website](https://commons.apache.org/proper/commons-lang) ·
[License: Apache-2.0](https://www.apache.org/licenses/LICENSE-2.0)

## Overview

Apache Commons Lang is a small library of static utility classes that fill gaps in
`java.lang`: null-safe string manipulation (`StringUtils`), array helpers
(`ArrayUtils`), reflection-free value builders (`EqualsBuilder`, `HashCodeBuilder`,
`ToStringBuilder`), tuples (`Pair`, `Triple`), and mutable primitive wrappers. It
began life inside Jakarta Commons around 2002 and is now one of the most
transitively depended-on artifacts in the entire Java ecosystem — present in the
dependency tree of most Spring, Hibernate, and Maven-plugin stacks whether or not
the application author added it directly.

The library's defining tension in 2026 is relevance erosion. Much of what made
`StringUtils` indispensable — `isBlank`, `strip`, `repeat` — has since landed in
the JDK itself (`String.isBlank()`, `String.strip()`, `String.repeat()` arrived in
Java 11)[^1]. On a modern JDK, a greenfield project can avoid Commons Lang
entirely for string work. What keeps it ubiquitous is inertia (it is already on
every classpath), its null-tolerant contract (`StringUtils.isBlank(null)` returns
`true` where `String.isBlank()` throws), and utilities the JDK still lacks:
consistent `hashCode`/`equals`/`toString` builders, `RandomStringUtils`, `Validate`
guard clauses, and `StopWatch`.

The current line is **commons-lang3** — artifact `org.apache.commons:commons-lang3`,
package `org.apache.commons.lang3`. The `3` is not cosmetic; it is the mechanism
that lets the old 2.x and the modern 3.x coexist on one classpath (see
Architecture). The project is actively maintained under the Apache Commons
umbrella, with releases tested against JDK 8, 11, 17, 21, and 25.

## Getting Started

Maven:

```xml
<dependency>
  <groupId>org.apache.commons</groupId>
  <artifactId>commons-lang3</artifactId>
  <version>3.20.0</version>
</dependency>
```

Gradle:

```groovy
implementation 'org.apache.commons:commons-lang3:3.20.0'
```

Typical usage — null-safe strings, guard clauses, and a generated `toString`:

```java
import org.apache.commons.lang3.StringUtils;
import org.apache.commons.lang3.Validate;
import org.apache.commons.lang3.builder.ToStringBuilder;
import org.apache.commons.lang3.builder.ToStringStyle;

String name = null;
StringUtils.isBlank(name);              // true — no NullPointerException
StringUtils.defaultIfBlank(name, "?");  // "?"
StringUtils.abbreviate("Commons Lang", 8); // "Comm..."

Validate.notNull(name, "name must not be null"); // throws NPE with message

// consistent, field-by-field toString without hand-writing it
new ToStringBuilder(this, ToStringStyle.JSON_STYLE)
    .append("id", 42)
    .append("name", "widget")
    .toString();                        // {"id":42,"name":"widget"}
```

## Architecture / How It Works

Commons Lang is deliberately a flat bag of stateless static-method utility
classes, not a framework. There is almost no shared runtime state, no
initialization, and no service abstraction. The design consequences are worth
understanding:

- **The `lang3` package rename is the load-bearing design decision.** When the
  library made incompatible changes for 3.0, it changed both the Maven artifactId
  (`commons-lang` → `commons-lang3`) and the Java package
  (`org.apache.commons.lang` → `...lang3`). Because the fully-qualified names
  differ, a single application can have both jars on the classpath without class
  clashes — an old transitive dependency pulling in 2.x does not collide with your
  3.x usage[^2]. This is a widely-cited example of how to ship a breaking major
  version for a foundational library.
- **Builders trade boilerplate for reflection risk.** `EqualsBuilder`,
  `HashCodeBuilder`, and `ToStringBuilder` have explicit `.append()` forms (no
  reflection) and `reflectionEquals` / `ReflectionToStringBuilder` variants that
  walk fields reflectively. The reflective variants are convenient but read private
  fields via `setAccessible`, which the Java Platform Module System (Java 9+) can
  block with `InaccessibleObjectException` when reflecting across module
  boundaries.
- **Null contract is the differentiator.** Every `StringUtils` method documents its
  behavior for `null` input and never throws NPE for it. This is the single most
  common reason teams keep the dependency even on Java 11+, where the JDK
  equivalents exist but reject `null`.
- **Scope has been actively trimmed.** Functionality that grew beyond "java.lang
  helpers" was spun out. `StringEscapeUtils`, `WordUtils`, and text-similarity code
  were deprecated in lang3 and relocated to the separate Apache Commons Text
  project around 3.6[^3]. This keeps the core small but means older tutorials point
  at classes that are now deprecated shells.

## Production Notes

**Version coexistence is a feature, but it hides bloat.** Because 2.x and 3.x can
both be present, a `mvn dependency:tree` will often show `commons-lang:commons-lang`
(the ancient 2.x) dragged in by some legacy transitive dependency alongside your
`commons-lang3`. Both ship. Audit and exclude the 2.x artifact if you do not
actually need it.

**`RandomStringUtils` default randomness is not cryptographically secure.** The
long-standing `RandomStringUtils.random(...)` methods draw from a shared
`java.util.Random`-style source and must never be used for tokens, passwords, or
secrets. Recent releases add explicit `RandomStringUtils.secure()` /
`.secureStrong()` factory methods backed by `SecureRandom`; prefer those for
anything security-relevant, and treat the plain static methods as deprecated for
that purpose[^4].

**Reflective builders and JPMS/modern GC.** `ReflectionToStringBuilder` and
`EqualsBuilder.reflectionEquals` do field-level reflection every call. On hot paths
this is measurably slower than hand-written or `.append()`-form code, and under the
module system they can fail outright when the target type lives in a module that
does not `opens` its package. Prefer explicit builders in library code.

**Deprecated-but-present surface.** Classes moved to Commons Text still exist in
lang3 as deprecated stubs for source compatibility. New code should depend on
`commons-text` for escaping (`StringEscapeUtils`) and word manipulation
(`WordUtils`) rather than the deprecated lang3 copies.

**Diminishing returns on modern JDKs.** On Java 17/21, a large fraction of
`StringUtils`/`ArrayUtils` usage can be replaced by `String`, `Objects`,
`Optional`, streams, and `List.of`. Keeping the dependency is fine (it is tiny and
stable), but new teams should not reach for it reflexively before checking whether
the JDK already covers the case.

**Stability.** Within the 3.x line the API is strongly backward compatible;
upgrades across minor versions rarely break callers. The one hard migration in the
project's history is 2.x → 3.x, which required package and artifact renames.

## When to Use / When Not

**Use when:**
- You need null-safe string/array/number helpers with a documented `null`
  contract, especially on Java 8.
- You want consistent `equals`/`hashCode`/`toString` without an annotation
  processor.
- The jar is already a transitive dependency and you want to use it directly
  rather than adding another utility library.
- You need `StopWatch`, `Validate` guard clauses, or `Pair`/`Triple` and don't want
  to hand-roll them.

**Avoid when:**
- You're on a modern JDK (11+) and the specific need is covered by
  `String`/`Objects`/`Optional`/streams — skip the dependency.
- You need cryptographically secure randomness and would be tempted by the
  non-secure `RandomStringUtils` defaults.
- You want a broader functional/collections toolkit — Guava is a fuller (heavier)
  answer.
- You want compile-time-generated `equals`/`toString` — Lombok or Java `record`s
  fit better than runtime builders.

## Alternatives

- google/guava — use instead when you want a single broad utility library
  (immutable collections, caching, functional helpers) and accept the larger jar
  and faster deprecation cadence.
- apache/commons-text — use instead of the deprecated lang3 stubs when you need
  string escaping, similarity, or word-manipulation utilities.
- apache/commons-collections — use for collection-specific utilities (bags,
  bidi-maps, transformers) that Lang deliberately does not cover.
- projectlombok/lombok — use instead of `EqualsBuilder`/`ToStringBuilder` when you
  want annotation-driven, compile-time-generated boilerplate.
- The JDK standard library — on Java 11+, prefer built-in `String.isBlank`/`strip`/
  `repeat`, `Objects.requireNonNull`/`equals`/`hash`, and records before adding a
  dependency.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2002 | Initial release as part of Jakarta Commons. |
| 2.0 | 2003 | Expanded `StringUtils`, builders, mutable/enum helpers. |
| 2.6 | 2011 | Final 2.x line (`org.apache.commons.lang`). |
| 3.0 | 2011 | Package + artifact rename to `lang3`; Java 5 baseline[^2]. |
| 3.6 | 2017 | Text/escape utilities deprecated, moved to Commons Text[^3]. |
| 3.12 | 2021 | Java 8 baseline; broad JDK 17 test coverage. |
| 3.20.0 | 2026 | Current release; tested on JDK 8/11/17/21/25 (per README). |

## References

[^1]: JDK 11 `String` API additions (`isBlank`, `strip`, `repeat`, `lines`). https://docs.oracle.com/en/java/javase/11/docs/api/java.base/java/lang/String.html
[^2]: Apache Commons Lang — "Upgrade from 2.x to 3.0" (package and artifact rename rationale). https://commons.apache.org/proper/commons-lang/article3_0.html
[^3]: Apache Commons Text — successor project for string escaping/similarity utilities relocated from Lang. https://commons.apache.org/proper/commons-text/
[^4]: Apache Commons Lang `RandomStringUtils` Javadoc — `secure()` / `secureStrong()` vs. non-secure defaults. https://commons.apache.org/proper/commons-lang/apidocs/org/apache/commons/lang3/RandomStringUtils.html

## Tags

java, jvm, utility-library, apache-commons, string-manipulation, null-safety, stdlib-extension, maven, backend, standard-library
