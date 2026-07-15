# thymeleaf/thymeleaf

> A server-side Java template engine whose templates are valid HTML files that render unchanged in a browser.

[GitHub repo](https://github.com/thymeleaf/thymeleaf) ·
[Official website](http://www.thymeleaf.org) ·
[License: Apache-2.0](https://github.com/thymeleaf/thymeleaf/blob/3.1-master/LICENSE.txt)

## Overview

Thymeleaf is a server-side template engine for the JVM, created by Daniel Fernández with a first release around 2011[^1]. Its defining idea is "natural templates": a template is a well-formed HTML document with extra `th:*` attributes, so it opens in a browser as a static prototype and also renders dynamically on the server. This distinguishes it from JSP, FreeMarker, and Velocity, whose templates are not valid standalone HTML. The tradeoff is verbosity — logic lives in XML attributes rather than in a compact tag syntax — and a rendering model that is interpreted, not compiled to bytecode.

In practice Thymeleaf's reach is almost entirely tied to Spring. After Spring stopped recommending JSP, Thymeleaf became the default view technology in most Spring Boot server-rendered tutorials and starters, via the `thymeleaf-spring6` (or `-spring5`) integration module[^2]. Outside the Spring world its adoption is thin; the engine works standalone, but the ergonomics, `SpringEL` expressions, and form/CSRF integration that make it pleasant are Spring-specific.

The central tension in 2026 is relevance. Thymeleaf is stable, well-documented, and maintained, but development has slowed and the server-rendered-HTML use case it serves is under pressure from SPA/API architectures on one side and from faster compiled Java template engines (JTE, Rocker) on the other. It remains a reasonable, boring default for Spring MVC pages — not a performance leader.

## Getting Started

Maven (Spring Boot pulls the right version transitively):

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

A template is ordinary HTML. Dynamic behavior is added through attributes in the `th` namespace:

```html
<!-- src/main/resources/templates/users.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<body>
  <h1 th:text="${title}">Placeholder Title</h1>
  <ul>
    <li th:each="user : ${users}" th:text="${user.name}">Sample name</li>
  </ul>
  <a th:href="@{/users/{id}(id=${current.id})}">Profile</a>
</body>
</html>
```

The literal text (`Placeholder Title`, `Sample name`) is what a designer sees when opening the file directly; at runtime the `th:*` attributes replace it. Expression prefixes carry distinct meaning: `${...}` variable, `*{...}` selection on a `th:object`, `#{...}` i18n message, `@{...}` context-relative URL, `~{...}` fragment.

## Architecture / How It Works

Thymeleaf 3.0 (2016) was a full rewrite that replaced the DOM-based processing of 2.x with an event-based engine[^3]. Instead of parsing a template into a mutable node tree, 3.0 parses it into a sequence of events (open tag, text, close tag) and streams them through processors. This cut memory use and improved throughput substantially over 2.x, and enabled non-HTML template modes.

Key concepts:

- **Template modes** — `HTML`, `XML`, `TEXT`, `JAVASCRIPT`, `CSS`, `RAW`. The same engine renders email bodies, plain-text, and inline JS/CSS, not just web pages.
- **Dialects and processors** — the Standard Dialect supplies `th:*` attribute processors. Behavior is extended by registering additional dialects; the engine itself is a processor-dispatch loop over the event stream.
- **Expression evaluation** — in a Spring app, `${...}` is evaluated by Spring Expression Language (SpEL); standalone it uses OGNL. Both are reflective and interpreted at request time unless results are cached.
- **Template resolution and caching** — parsed templates are cached by name. This is the single most important production setting: parsing on every request is expensive, so the cache must be on in production.
- **Fragments** — `th:fragment` / `th:insert` / `th:replace` with `~{...}` expressions provide composition. Whole-page layout inheritance is not core; the widely used `nz.net.ultraq.thymeleaf` layout-dialect is a third-party add-on.
- **Decoupled logic** — since 3.0, `th:*` logic can live in a separate `.th.xml` file, keeping the HTML completely free of Thymeleaf attributes for pure design fidelity.

The engine is interpreted end to end: templates are never compiled to Java bytecode. Correctness and flexibility are high; raw rendering speed is not the design goal.

## Production Notes

**Enable template caching.** Spring Boot disables the cache in development for hot reload (`spring.thymeleaf.cache=false`). If that leaks into production, every request re-parses the template and re-resolves expressions — a large, silent throughput hit. Confirm `spring.thymeleaf.cache=true` (the production default) in your deployed profile.

**The 3.1 web-context break.** Thymeleaf 3.1 (2023) removed direct access to the servlet API from expressions: `${#request}`, `${#session}`, `${#servletContext}`, and `${#response}` were deleted for security and portability reasons[^4]. Apps that read request/session state inside templates — a common pattern — will fail on upgrade and must move that data into the model or into utility beans. This is the most disruptive migration Thymeleaf has shipped and is the reason many codebases stayed on 3.0.

**Escaping and XSS.** `th:text` HTML-escapes by default, which is correct. `th:utext` emits raw, unescaped output; any user-controlled data passed through `th:utext` is a stored-XSS vector. Audit `utext` usage. Inlined expressions `[[...]]` escape, `[(...)]` do not.

**Performance ceiling.** Thymeleaf's interpreted, reflection-heavy rendering is meaningfully slower than compiled engines like JTE or Rocker, which generate Java classes and are validated against the Java compiler. For pages rendered at very high request rates, benchmark before assuming Thymeleaf is fast enough; the natural-template ergonomics come at a runtime cost.

**Version alignment with Spring.** Use `thymeleaf-spring6` with Spring 6 / Spring Boot 3 (Jakarta EE namespaces) and `thymeleaf-spring5` with Spring 5 (javax). Mixing the integration module against the wrong Spring generation produces obscure `NoClassDefFoundError`/namespace failures. Let Spring Boot's BOM pick versions rather than pinning by hand.

**Fragment and expression overhead.** Deeply nested fragments plus complex SpEL expressions evaluated per row in large `th:each` loops are a common source of slow pages. Precompute in the controller; keep template expressions shallow.

## When to Use / When Not

**Use when:**
- You are building server-rendered pages in Spring MVC / Spring Boot and want the supported, best-documented view layer.
- Designers and developers share HTML files and you value templates that preview statically without a running server.
- You render mixed output (HTML pages, HTML/text emails) and want one engine for all of it.
- Throughput is moderate and developer ergonomics matter more than last-millisecond rendering speed.

**Avoid when:**
- You are outside Spring — the standalone experience loses most of the integration that makes Thymeleaf worthwhile.
- You need maximum rendering throughput — a compiled engine (JTE, Rocker) will beat it.
- Your frontend is a SPA or you serve JSON only — a template engine is the wrong layer entirely.
- You cannot absorb the 3.1 migration and depend on servlet objects inside templates.

## Alternatives

- jknack/handlebars.java — logic-less Handlebars templates on the JVM; use when you want a portable, front-end-familiar syntax rather than HTML attributes.
- casid/jte — compiled, type-safe Java/Kotlin templates; use when rendering speed and compile-time checking outrank natural-template previewing.
- fizzed/rocker — compiled, statically typed templates; use when you want near-hand-written render performance.
- apache/freemarker — mature, feature-rich general-purpose template language; use when you need templating beyond HTML with a compact tag syntax and don't need browser-openable templates.
- PebbleTemplates/pebble — Twig-inspired engine with inheritance and autoescaping; use when you prefer a `{{ }}`/`{% %}` syntax with first-class layout inheritance built in.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2011 | Initial public release; natural-templating concept, DOM-based engine[^1]. |
| 2.0 | 2012 | Dialect model matured; Spring integration established. |
| 2.1 | 2014 | Last 2.x line; text template modes limited. |
| 3.0 | 2016-05 | Full rewrite to an event-based engine; new template modes, decoupled logic, big performance gain over 2.x[^3]. |
| 3.1 | 2023 | Removed `#request`/`#session`/`#servletContext`/`#response` from expressions; `thymeleaf-spring6` for Spring 6 / Jakarta[^4]. |

## References

[^1]: Thymeleaf project site and documentation. https://www.thymeleaf.org/documentation.html
[^2]: Spring Boot reference, "Template Engines" — Thymeleaf as the recommended server-side option. https://docs.spring.io/spring-boot/reference/web/servlet.html#web.servlet.spring-mvc.template-engines
[^3]: "What's new in Thymeleaf 3.0" — event-based engine rewrite and new template modes. https://www.thymeleaf.org/doc/articles/thymeleaf3migration.html
[^4]: "Thymeleaf 3.1: What's new and how to migrate" — removal of web/servlet API access from expressions. https://www.thymeleaf.org/doc/articles/thymeleaf31whatsnew.html

## Tags

java, template-engine, server-side-rendering, spring, spring-boot, html, mvc, web, jvm, view-layer
