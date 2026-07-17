# HubSpot/jinjava

> A Java implementation of the Jinja template syntax — the engine behind HubSpot's HubL and CMS rendering.

[GitHub repo](https://github.com/HubSpot/jinjava) ·
[License: Apache-2.0](https://github.com/HubSpot/jinjava/blob/master/LICENSE)

## Overview

Jinjava is a server-side template engine for Java that renders a large subset of the Jinja/Jinja2 syntax familiar from Python[^1]. HubSpot built it because its CMS templating language, HubL, is Jinja-flavored, and the platform runs on the JVM — there was no faithful Java engine for that dialect, so they forked and rewrote the abandoned `jangod` Django-template port[^2]. It has been maintained continuously since 2014 and, per the README, renders "thousands of websites with hundreds of millions of page views per month" on the HubSpot CMS[^1].

The defining characteristic is that Jinjava is **not** a general-purpose port of Python Jinja2. It implements the subset of tags, filters, and functions HubSpot needs, plus HubSpot-specific extensions, and diverges from CPython Jinja2 in expression semantics and edge-case behavior. Teams that adopt it expecting drop-in parity with the Python engine will hit gaps; teams that adopt it to render HubL (or a Jinja-like DSL of their own) get an extensible, production-hardened interpreter.

It is a niche but load-bearing library: modest star count, but embedded deep in one of the larger commercial CMS platforms and pulled into any JVM project that needs Jinja-style templating. Development activity tracks HubSpot's own needs rather than community feature requests.

## Getting Started

Add the Maven dependency (published to Maven Central under `com.hubspot.jinjava`):

```xml
<dependency>
  <groupId>com.hubspot.jinjava</groupId>
  <artifactId>jinjava</artifactId>
  <version>{ LATEST_VERSION }</version>
</dependency>
```

Requires Java 8 or newer[^1]. Minimal render:

```java
Jinjava jinjava = new Jinjava();

Map<String, Object> context = new HashMap<>();
context.put("name", "Jared");

String template = "<div>Hello, {{ name }}!</div>";
String rendered = jinjava.render(template, context);
// -> <div>Hello, Jared!</div>
```

A `Jinjava` instance is reusable and intended to be shared; per-render state lives in a short-lived `JinjavaInterpreter`.

## Architecture / How It Works

Jinjava is a tree-walking **interpreter**, not a compiler. A render is roughly: tokenize the template into a node tree (text nodes, tag nodes, expression nodes), then walk the tree against a `Context` and emit a string. There is no compilation to Java bytecode or caching of a compiled form — each `render()` reparses the template text.

Key internals:

- **Expression evaluation** is delegated to a Java Expression Language (EL) engine (JUEL-derived), which is why `{{ ... }}` expressions and filter chaining behave like EL under the hood rather than like Python. This is the main source of subtle divergence from CPython Jinja2.
- **`Context`** is a scoped, map-backed symbol table. Scopes are pushed and popped as the interpreter enters blocks (loops, includes, macros), so variable shadowing follows lexical block nesting.
- **`ResourceLocator`** abstracts template loading for tags like `{% extends %}` and `{% include %}`. The default `ClasspathResourceLocator` resolves from the classpath; `FileResourceLocator`, `CascadingResourceLocator`, and custom implementations plug in via `jinjava.setResourceLocator(...)`[^1].
- **Extensibility** is via the global context: `registerTag`, `registerFilter`, and `registerFunction` (the latter binding a public static Java method to a template-callable function)[^1]. Built-in libraries live under `com.hubspot.jinjava.lib`.
- **Execution modes.** Beyond straight rendering, Jinjava supports a deferred/eager execution mode used by HubSpot to partially render templates when some values are not yet known, preserving the un-renderable parts as source for a later pass. This is unusual for a template engine and reflects HubSpot's editor/preview pipeline rather than a generic need.

Configuration (output size limits, rendering depth, error handling strategy, locale, execution mode) flows through an immutable `JinjavaConfig` built with a builder and passed to the `Jinjava` constructor.

## Production Notes

**Reparse cost.** Because templates are interpreted from source on every render with no compiled-form cache, hot templates rendered at high volume pay tokenization + tree-build cost each time. If you render the same templates repeatedly, cache the rendered output or the parsed structure at your application layer; do not assume the engine memoizes anything for you.

**Template injection is a real risk.** The README explicitly warns that adding a `FileResourceLocator` lets user-controlled template input reach the filesystem — e.g. `{% include '/etc/passwd' %}`[^1]. Treat any template whose source is influenced by end users as untrusted: keep the default classpath locator, use `JinjavaConfig` limits (max output size, max rendering depth) to bound resource exhaustion, and review which built-in filters/functions expose reflection or I/O. Server-side template injection (SSTI) applies here the same way it does to Freemarker and Velocity.

**Jinja2 parity gaps.** This is the most common surprise. Whitespace control, some filters, exception/undefined-variable behavior, and expression edge cases differ from CPython Jinja2. Do not port Python Jinja2 templates verbatim and expect identical output — test the specific constructs you use. The engine targets HubL, so HubL semantics win where they conflict with upstream Jinja.

**Error handling.** Rendering collects errors rather than always throwing; `RenderResult` carries the output plus a list of `TemplateError`s. Decide explicitly whether your integration fails closed (reject output with errors) or fails open (serve partial output) — the default is permissive and will happily emit a partially-rendered page.

**Java version and dependencies.** Java 8 is the floor. Jinjava pulls in Guava and an EL implementation transitively; version conflicts with an app's own Guava are the usual dependency-hell complaint. A legacy `2.0.11-java7` artifact exists for Java 7 holdouts but is not maintained[^1].

**Concurrency.** Share one `Jinjava` instance across threads; the per-render `JinjavaInterpreter` and `Context` are not meant to be shared between concurrent renders.

## When to Use / When Not

**Use when:**
- You need to render HubL or HubSpot-CMS-style templates on the JVM.
- You want Jinja/Django-style syntax in a Java service and value HubSpot's production hardening.
- You need custom tags/filters/functions exposed to a Jinja-like DSL for non-Java authors.
- You want a template layer with configurable resource limits and deferred/partial rendering.

**Avoid when:**
- You need exact CPython Jinja2 parity — use the real Jinja from Python, or expect to test around divergences.
- You render fully-static or Java-authored views where a compiled engine (Freemarker, Thymeleaf) gives better performance and IDE support.
- Your templates come from untrusted users and you cannot invest in SSTI hardening.
- You want a large third-party ecosystem of tags/filters — Jinjava's built-ins are scoped to what HubSpot needs.

## Alternatives

- pallets/jinja — the canonical Python Jinja2; use it when your stack is Python and you want the reference semantics.
- apache/freemarker — mature JVM template engine with its own syntax; use it when you want a compiled, well-tooled Java-native engine and don't need Jinja syntax.
- thymeleaf/thymeleaf — HTML-first natural templating for Spring; use it when you want valid-HTML templates and tight Spring MVC integration.
- apache/velocity-engine — older JVM template engine; use it for simple, long-stable templating where VTL is acceptable.
- spullara/mustache.java — logic-less Mustache for Java; use it when you want strict template/logic separation and cross-language template reuse.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2014-10 | Repository created; fork/rewrite of the abandoned `jangod` Django-template port[^2]. |
| 2.0.x | — | 2.x series on Maven Central; split `-java7` classifier for Java 7 users, mainline on Java 8[^1]. |
| current | 2026-07 | Actively maintained by HubSpot; ongoing commits tracking CMS/HubL needs. |

## References

[^1]: HubSpot/jinjava README — install, usage, template loading, extensibility, and the Java 8 requirement / Java 7 legacy artifact. https://github.com/HubSpot/jinjava/blob/master/README.md
[^2]: `jangod`, the original Django-template Java port Jinjava was forked from (archived Google Code project). https://code.google.com/p/jangod/
[^3]: Javadocs for `com.hubspot.jinjava`. https://www.javadoc.io/doc/com.hubspot.jinjava/jinjava

## Tags

java, template-engine, jinja, jinja2, hubspot, hubl, jvm, server-side-rendering, apache-2.0, templating
