# PebbleTemplates/pebble

> A Java template engine modeled on Twig — inheritance, autoescaping, and a readable `{{ }}` / `{% %}` syntax, evaluated by an interpreter rather than compiled to bytecode.

[GitHub repo](https://github.com/PebbleTemplates/pebble) ·
[Official website](https://pebbletemplates.io) ·
[License: BSD-3-Clause](https://github.com/PebbleTemplates/pebble/blob/master/LICENSE)

## Overview

Pebble is a server-side Java templating engine originally written by Mitchell Bösecke around 2013[^1], now maintained under the PebbleTemplates organization. Its design is lifted almost directly from Twig, the template engine behind Symfony: the same `{{ variable }}` output tags, `{% tag %}` control tags, `{# comment #}` comments, filters via the `|` pipe, and — the feature the project foregrounds — template inheritance through `{% extends %}` and `{% block %}`. Developers coming from Twig, Jinja2, Django templates, or Liquid will find the syntax immediately familiar.

The engine positions itself against Java's older incumbents (FreeMarker, Velocity) and against Thymeleaf, Spring's default. Its two stated differentiators are inheritance and readability; its two structural commitments are autoescaping on by default (HTML-escaped output unless a value is explicitly marked raw) and built-in internationalization via an `i18n` function backed by `ResourceBundle`. It is deliberately a *presentation* template engine — it discourages arbitrary logic in templates, so there is no way to call setters or mutate application state from a template, only to read and format data passed in through the context.

The defining tension is scope. Pebble is small, sandboxed, and opinionated, which makes it safe and predictable but also means it will never match FreeMarker's raw feature surface or Thymeleaf's HTML-native "natural templating" (templates that render as valid HTML in a browser without processing). It is a good fit for teams who want Twig ergonomics on the JVM and are willing to keep logic out of their views.

## Getting Started

Maven, using the current `io.pebbletemplates` coordinates:

```xml
<dependency>
  <groupId>io.pebbletemplates</groupId>
  <artifactId>pebble</artifactId>
  <version>3.2.2</version>
</dependency>
```

Templates live on the classpath (default suffix `.peb`) and are resolved by a `ClasspathLoader`:

```java
// home.peb:
//   {% extends "base.peb" %}
//   {% block content %}Hello, {{ name | upper }}!{% endblock %}

PebbleEngine engine = new PebbleEngine.Builder().build();
PebbleTemplate template = engine.getTemplate("home.peb");

Map<String, Object> context = new HashMap<>();
context.put("name", "world");

Writer writer = new StringWriter();
template.evaluate(writer, context);
System.out.println(writer.toString());   // Hello, WORLD!
```

A `PebbleEngine` is thread-safe and meant to be built once and reused; `getTemplate` is cached, so repeated calls with the same name do not re-parse.

## Architecture / How It Works

Pebble is an **interpreter, not a compiler**. A template string passes through a lexer that emits tokens, a parser that builds an abstract syntax tree of `Node` objects, and evaluation walks that tree, writing to the supplied `Writer` against a per-render context. It does not generate or compile Java source/bytecode the way some engines (e.g. JTE, Rocker) do, so there is no build-time code generation step — the tradeoff is per-node interpretation overhead at render time, offset by caching.

Caching happens at two levels: a **template cache** (parsed template objects, keyed by name) and an optional **tag cache** for `{% cache %}` blocks. Both are pluggable through the builder; the default is an in-memory cache. Parsing is the expensive part, so in production almost all cost is paid once at first render of each template.

Template resolution is the other core subsystem. **Loaders** map a template name to source: `ClasspathLoader`, `FileLoader`, `StringLoader`, and `ServletLoader`, plus a `DelegatingLoader` that chains several. As of 4.1.x the default when no loader is configured is a bare `ClasspathLoader`; earlier versions defaulted to a `DelegatingLoader` (classpath + filesystem), and `FileLoader` now requires an explicit base directory[^2]. This is the most common source of "template not found" breakage on upgrade.

Extensibility runs through the `Extension` interface: you register custom **filters**, **functions**, **tests**, **operators**, and **tags**. Autoescaping is implemented as an escaper extension with per-context escaping strategies (`html`, `js`, `css`, `url_param`), and the `{% autoescape %}` tag toggles it for a region. The Spring Boot integration is shipped as separate starter artifacts that wire a `PebbleEngine` into Spring MVC and WebFlux view resolution.

## Production Notes

- **Coordinate and package migration.** The package was renamed from `com.mitchellbosecke` to `io.pebbletemplates` in 3.2.x[^3], and the Maven groupId moved accordingly. Any code importing the old package, and any build pulling the old coordinates, breaks on that upgrade. This is the single biggest gotcha in the project's history.
- **Loader default change (4.1.x).** Dropping the implicit `DelegatingLoader` means filesystem templates that used to resolve "for free" no longer do; you must configure a `FileLoader` with a base directory explicitly[^2]. Templates on the classpath are unaffected.
- **Spring Boot starter split (4.0.x).** There are two starters — `pebble-spring-boot-starter` for Spring Boot 4.x and `pebble-legacy-spring-boot-starter` for 3.x — and a batch of `pebble.*` properties moved under `pebble.servlet.*` / `pebble.reactive.*`[^4]. Picking the wrong starter for your Boot version, or leaving old flat property names in `application.yml`, silently drops configuration.
- **Autoescaping is HTML by default, and it is your safety net.** Turning it off globally, or over-using the `raw` filter, reintroduces XSS exposure. Prefer marking individual trusted values raw over disabling escaping for a block.
- **No logic escape hatch is a feature, not a bug.** You cannot call arbitrary methods with side effects from a template. Data shaping belongs in the controller/service layer that builds the context; fighting this by cramming logic into macros tends to produce unreadable templates.
- **Caching in dev.** The template cache will happily serve a stale parse; disable it (or use `StringLoader`/no cache) during template development, and keep it on in production.
- **Ecosystem size.** With roughly 1.2k stars this is a modestly-sized, community-maintained project, not a large-vendor product. Release cadence is steady but not fast; evaluate the issue tracker and recent commit activity before betting a large system on niche features.

## When to Use / When Not

**Use when:**
- You want Twig/Jinja-style syntax and template inheritance on the JVM.
- You value a sandboxed, logic-light view layer with autoescaping on by default.
- You're on Spring Boot and want a drop-in view engine via the official starter.
- Internationalization through `ResourceBundle` is a first-class need.

**Avoid when:**
- You need natural templating (views that are valid, previewable HTML unprocessed) — Thymeleaf is built for that.
- You want maximum raw expressiveness and a huge built-in directive set — FreeMarker covers more.
- You want compiled, type-checked templates with build-time codegen and near-zero render overhead — JTE or Rocker fit better.
- Your rendering is client-side; Pebble is server-side only.

## Alternatives

- thymeleaf/thymeleaf — use instead when you want natural templating and tight Spring integration as the default choice.
- apache/freemarker — use instead when you need a broader built-in feature set and macro power and don't mind heavier syntax.
- casid/jte — use instead when you want compiled, statically-typed templates with minimal runtime cost.
- fizzed/rocker — use instead when you want hot-reloadable, compiled, near-zero-overhead Java templates.
- spullara/mustache.java — use instead when you want strict logic-less templates portable across Mustache implementations.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | ~2013 | Created by Mitchell Bösecke; repo on GitHub since 2012-12-27[^1]. |
| 3.2.x | — | Package rename `com.mitchellbosecke` → `io.pebbletemplates`; default suffix changed to `.peb`; `getInstance` → `createInstance` on `BinaryOperator`[^3]. |
| 4.0.x | — | Spring Boot 4 support; separate legacy starter for Boot 3.x; `pebble.*` properties moved under `.servlet` / `.reactive`[^4]. |
| 4.1.x | — | Default loader is now a bare `ClasspathLoader`; `FileLoader` requires an explicit base directory[^2]. |

## References

[^1]: Pebble README license header, "Copyright (c) 2013, Mitchell Bösecke"; GitHub repository created 2012-12-27. https://github.com/PebbleTemplates/pebble
[^2]: Pebble README, "Breaking changes in version 4.1.x". https://github.com/PebbleTemplates/pebble#breaking-changes-in-version-41x
[^3]: Pebble README, "Breaking changes in version 3.2.x". https://github.com/PebbleTemplates/pebble#breaking-changes-in-version-32x
[^4]: Pebble README, "Breaking changes in version 4.0.x", and Spring Boot integration guide. https://pebbletemplates.io/wiki/guide/spring-boot-integration/

## Tags

java, template-engine, twig, server-side-rendering, jvm, spring-boot, html, autoescaping, i18n, view-layer
