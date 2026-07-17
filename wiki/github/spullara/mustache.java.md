# spullara/mustache.java

> A dependency-free Java implementation of the logic-less Mustache templating language, with optional concurrent evaluation.

[GitHub repo](https://github.com/spullara/mustache.java) ·
[API docs](http://spullara.github.io/mustache/apidocs/) ·
License: Apache-2.0[^1]

## Overview

Mustache.java is a Java port of the Mustache templating spec, originally
derived from mustache.js[^2]. Mustache is deliberately "logic-less": templates
contain variable interpolations (`{{name}}`), sections (`{{#items}}...{{/items}}`),
inverted sections, partials, and comments, but no arbitrary expressions or
control flow. The philosophy pushes presentation logic back into the host
language, which keeps templates readable and portable across the many Mustache
implementations. This library passes the shared Mustache specification test
suite modulo whitespace differences[^2].

The project dates to 2010 and was written by Sam Pullara (originally at
RightTime, Inc.)[^1]. Its defining production credential is Twitter, which used
it to render the website, email, and syndicated widgets[^2] — a workload that
shaped its performance-first design. The compiler jar is roughly 100k with no
external runtime dependencies, which makes it easy to embed.

The library's most distinctive feature, and its central tradeoff, is optional
concurrent evaluation. Any scope value that returns a `Callable` can be executed
on a supplied `ExecutorService`, letting independent parts of a template render
in parallel or stream as they complete. This buys throughput at the cost of a
more subtle execution model than the strictly serial evaluation most templating
engines offer. It is also the biggest behavioral divergence from mustache.js[^2].

## Getting Started

Maven (Java 8+); the `compiler` module covers most use cases:

```xml
<dependency>
  <groupId>com.github.spullara.mustache.java</groupId>
  <artifactId>compiler</artifactId>
  <version>0.9.14</version>
</dependency>
```

```java
import com.github.mustachejava.DefaultMustacheFactory;
import com.github.mustachejava.Mustache;
import com.github.mustachejava.MustacheFactory;
import java.io.*;
import java.util.*;

public class Example {
  public static void main(String[] args) throws IOException {
    MustacheFactory mf = new DefaultMustacheFactory();
    Mustache m = mf.compile(new StringReader("{{name}}, {{feature.description}}!"),
                            "example");
    Map<String, Object> scope = new HashMap<>();
    scope.put("name", "Mustache");
    scope.put("feature", Map.of("description", "Perfect"));
    Writer w = new OutputStreamWriter(System.out);
    m.execute(w, scope).flush();   // -> Mustache, Perfect!
  }
}
```

The Java 6/7 line is frozen at the 0.8.x branch; 0.9.0 and later require Java 8[^2].

## Architecture / How It Works

A `MustacheFactory` compiles a template into a tree of `Code` nodes; `Mustache.execute`
walks that tree against an array of scopes, writing to a `Writer`. Scope
resolution is reflective: for each tag the engine tries `Map` keys, then
non-private fields, then no-arg methods on the objects in the scope stack, and
caches the resolved accessor per class as a "wrapper" so repeated renders skip
the reflection cost[^2]. Any `Iterable` drives section iteration.

The codebase is split across Maven modules. `compiler` is the core parser and
renderer. `codegen` generates Java source for guards and mustache nodes, and the
`indy` module builds on it to compile templates down to bytecode via
invokedynamic — an optional path whose payoff is described as application-specific
rather than universal[^2]. The `handlebar` module is a small server that renders
templates against JSON for designer mockups, and `example` is an end-to-end demo.

Extension points are broad. Lambdas map to Java 8 `Function` for post-substitution
transforms, or `TemplateFunction` when you want the output reparsed as Mustache
(pre-substitution). Template inheritance is supported via the `{{<parent}}` /
`{{$block}}` syntax from the spec's inheritance discussion[^2]. Almost every stage
of compilation and rendering is pluggable through a custom `MustacheVisitor`;
`CapturingMustacheVisitor` can extract live sample data for building test mocks,
and an `invert` operation solves for the data that would produce a given rendered
string from a template.

## Production Notes

- **Templates are trusted input by default.** The README states plainly that the
  library is UNSAFE for untrusted templates[^3]. Use `SafeMustacheFactory` and
  whitelist every allowed template and partial name if template sources are not
  fully under your control — partial resolution can otherwise reach arbitrary
  classpath or filesystem paths.
- **Reflective access is a coupling risk.** Because tags resolve to fields and
  methods by name, renaming a getter or field silently changes template output,
  and obfuscation/minification tools or a strict Java module system
  (`--add-opens`) can break reflective access at runtime with no compile-time
  warning.
- **Concurrent evaluation needs an explicit `ExecutorService`.** Returning
  `Callable` values only parallelizes if you configured an executor on the
  factory; otherwise those callbacks run serially and any blocking work stalls
  the whole render. Sizing and lifecycle of that pool are your responsibility.
- **Reuse compiled `Mustache` instances.** Compilation is far more expensive than
  execution; compile once and render many times. `MustacheFactory` caches
  partials, so long-lived factories are the intended usage.
- **Maintenance is quiet.** The project is mature and stable rather than actively
  developed — the most recent commit activity is from 2024[^1], with a long tail
  of point releases on the 0.9.x line. Treat it as feature-complete; do not expect
  rapid response to new issues.

## When to Use / When Not

**Use when:**
- You want a small, dependency-free, spec-compliant Mustache engine on the JVM.
- Your templates come from trusted sources (your own repo, designers you control).
- You benefit from parallel or streaming rendering of independent template regions.
- You value cross-language template portability with JS/Ruby/Go Mustache stacks.

**Avoid when:**
- You need in-template logic, filters, or expressions — pick a richer engine.
- You must render user-submitted templates without a strict whitelist.
- You want an actively evolving project with frequent releases and support.

## Alternatives

- jknack/handlebars.java — Handlebars superset of Mustache with helpers and richer expressions; use when logic-less is too restrictive.
- samskivert/jmustache — even smaller zero-dependency Mustache engine; use when you want minimal surface and no concurrency machinery.
- thymeleaf/thymeleaf — HTML-native templating with Spring integration; use for server-rendered web pages that need in-template logic.
- apache/velocity — mature general-purpose Java template engine; use when you want a scripting-style template language, not logic-less.
- trimou/trimou — Mustache/Handlebars engine with a helper ecosystem; use when you want Mustache syntax plus extensibility.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2010 | First release; derived from mustache.js, used at Twitter[^1][^2]. |
| 0.8.x | — | Last line supporting Java 6/7[^2]. |
| 0.9.0 | — | Java 8 minimum; lambdas via `Function`/`TemplateFunction`[^2]. |
| 0.9.10 | — | Version cited in current README docs[^2]. |
| 0.9.14 | 2024 | Latest tagged release[^1]. |

## References

[^1]: spullara/mustache.java repository and license (Apache-2.0; GitHub reports the SPDX as NOASSERTION due to a non-standard header, but the LICENSE file is the Apache License 2.0). https://github.com/spullara/mustache.java
[^2]: mustache.java README — features, modules, performance notes, Twitter deployment. https://github.com/spullara/mustache.java/blob/main/README.md
[^3]: mustache.java README security warning on untrusted templates and `SafeMustacheFactory`. https://github.com/spullara/mustache.java/blob/main/README.md

## Tags

java, templating, mustache, logic-less, jvm, text-rendering, template-engine, server-side, apache-2.0
