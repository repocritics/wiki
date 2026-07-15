# apache/freemarker

> A Java template engine for generating text output (HTML, config, source code, email) from a template plus a data model, defined by a two-decade obsession with backward compatibility.

[GitHub repo](https://github.com/apache/freemarker) ·
[Official website](https://freemarker.apache.org/) ·
[License: Apache-2.0](https://github.com/apache/freemarker/blob/2.3-gae/LICENSE)

## Overview

FreeMarker is a server-side template engine: you give it a template written in FreeMarker Template Language (FTL) and a data model (a tree of Java objects), and it renders text. It is a library embedded into a program, not an application — the canonical use is the View in a servlet-based MVC web app, but it is equally used to generate emails, configuration files, SQL, and source code. It has no required runtime dependencies and targets Java SE 8 as a minimum[^1].

The project predates its Apache home by well over a decade: FreeMarker started on SourceForge around 1999, and the 2.x line was maintained for years mainly by Dániel Dékány and Attila Szegedi before it entered the Apache Incubator in 2015 and graduated to a top-level project in 2018[^2]. That history is the single most important thing to understand about it: the entire active codebase is still the `2.3.x` line, and its defining engineering value is *not* breaking templates written fifteen years ago.

The defining tension is that backward-compatibility discipline against correctness. Rather than fix behavioral bugs outright (which would silently change old templates' output), FreeMarker gates fixes behind an `incompatible_improvements` version number — you opt into a batch of corrected behaviors by declaring the version you tested against. This keeps upgrades safe but means a default configuration runs with decade-old semantics unless you explicitly move the flag forward.

## Getting Started

Maven (the primary published artifact is the "gae" build under group `org.freemarker`):

```xml
<dependency>
  <groupId>org.freemarker</groupId>
  <artifactId>freemarker-gae</artifactId>
  <version>2.3.34</version>
</dependency>
```

```java
import freemarker.template.*;
import java.io.*;
import java.util.*;

// Build ONE Configuration for the whole application, then reuse it.
Configuration cfg = new Configuration(Configuration.VERSION_2_3_34);
cfg.setClassLoaderForTemplateLoading(App.class.getClassLoader(), "/templates");
cfg.setDefaultEncoding("UTF-8");
cfg.setTemplateExceptionHandler(TemplateExceptionHandler.RETHROW_HANDLER);
cfg.setRecognizeStandardFileExtensions(true); // .ftlh/.ftlx => auto-escaping

Map<String, Object> model = new HashMap<>();
model.put("user", "Tom");
model.put("items", List.of("a", "b", "c"));

Template t = cfg.getTemplate("hello.ftlh");   // .ftlh => HTML output format
t.process(model, new OutputStreamWriter(System.out));
```

```ftl
<#-- hello.ftlh -->
<h1>Hello, ${user}!</h1>
<ul>
  <#list items as item>
    <li>${item?upper_case}</li>
  <#else>
    <li>no items</li>
  </#list>
</ul>
```

## Architecture / How It Works

The three moving parts are `Configuration`, `Template`, and the *data model*.

- **`Configuration`** is the shared, application-scoped singleton. It holds the `TemplateLoader` (class-path, file-system, string, or a composite), the global settings (locale, number/date formats, output format, encoding), and the template cache. It is safe for concurrent template processing once fully configured, but you are not meant to mutate its settings after other threads start using it.
- **Template loading and caching.** `getTemplate()` parses source into an internal AST once and caches it. The cache re-checks the backing store on a delay (`template_update_delay`, historically five seconds) rather than on every request, which is why local template edits can appear not to take effect immediately. There is no bytecode generation — templates are interpreted from the AST.
- **The data model and `ObjectWrapper`.** FTL never sees Java objects directly; every value is adapted to a `TemplateModel` interface by an `ObjectWrapper`. `DefaultObjectWrapper` exposes `Map` as hashes, `List`/array as sequences, and JavaBean properties via getters. Underneath sits `BeansWrapper`, which reflectively exposes bean *methods* as well — powerful, and the root of FreeMarker's security surface (see below).

FTL itself has three syntactic surfaces: interpolations `${...}`, FTL directives `<#if>` / `<#list>` / `<#assign>` / `<#macro>`, and user-defined directives (macros and Java `TemplateDirectiveModel`s) invoked with `<@...>`. Value transformations use *built-ins* via the `?` syntax (`name?upper_case`, `seq?size`, `x!0` for a default, `x??` for existence). The language is deliberately not general-purpose: there is no arbitrary Java execution from FTL by design — only what the data model and configured built-ins expose.

## Production Notes

**Auto-escaping is opt-in by history, not by default.** A plain-text template does no HTML escaping, so `${userInput}` is an XSS hole. The fix is *output formats* (added in 2.3.24, 2016): use the `.ftlh` (HTML) / `.ftlx` (XML) extensions with `recognizeStandardFileExtensions`, set a default output format on the `Configuration`, or declare `<#ftl output_format="HTML">` per template. Auditing that every template renders through an escaping output format is a real, recurring task on inherited FreeMarker codebases.

**Server-side template injection (SSTI) is the marquee risk.** If untrusted users can supply or edit template *source*, `BeansWrapper` reflection plus built-ins like `?new` allow reaching arbitrary classes. The classic payload `<#assign x="freemarker.template.utility.Execute"?new()>${x("id")}` yields remote code execution. Never let end users author FTL. If you must, restrict reachable classes with `new_builtin_class_resolver` (`TemplateClassResolver.ALLOWS_NOTHING_RESOLVER`) and lock down the `ObjectWrapper`'s method-exposure level.

**`incompatible_improvements` is the upgrade lever.** Because FreeMarker will not change behavior silently, bumping the jar version alone rarely changes rendering. You get the fixes by raising the version passed to `new Configuration(...)` and re-testing. Treat this as a first-class part of any upgrade, and read the release notes for the specific improvements each version gates[^3].

**Configuration lifecycle footguns.** Constructing a `Configuration` per request is a common performance mistake — it discards the template cache and re-parses everything. Build one, hold it, share it. Conversely, mutating settings on a shared `Configuration` at runtime is a data race.

**Error handling.** By default an undefined or null variable is an error, not an empty string — you must use `!` (default) or `??` (existence) explicitly. In production set `TemplateExceptionHandler.RETHROW_HANDLER` and handle failures upstream; the default handler writes error detail into the output stream, which is not what you want facing users.

**FreeMarker 3 never landed.** A ground-up redesign (package `org.apache.freemarker`, separate artifacts) was attempted but stalled at alpha; there is no supported migration off 2.3, and effectively none is needed — 2.3.x remains the maintained line. Plan long-term on staying there.

## When to Use / When Not

**Use when:**
- You need a mature, dependency-free Java template engine for HTML, email, or code/config generation with predictable, well-documented semantics.
- Backward compatibility matters — you have templates that must keep rendering identically across years of dependency upgrades.
- You want rich formatting control (locale-aware number/date built-ins, custom formats, macros) beyond what a logic-less engine offers.

**Avoid when:**
- Untrusted users author templates and you cannot fully sandbox them — the SSTI surface makes this genuinely dangerous.
- You want HTML-valid, designer-editable templates that render in a browser as-is — Thymeleaf's natural templating fits better.
- You want a logic-less, cross-language template contract shared with non-JVM services — a Mustache/Handlebars-style engine is a better fit.
- You are building a modern SPA/SSR frontend — server-rendered FTL is not that world.

## Alternatives

- thymeleaf/thymeleaf — use instead when you want natural, HTML-valid templates and Spring Boot's default view engine with escaping on by default.
- apache/velocity-engine — use instead when you want an even simpler, older VTL syntax and minimal surface; note far lower development activity.
- PebbleTemplates/pebble — use instead when you want Twig/Jinja-like syntax with autoescaping on by default and inheritance-based layouts.
- spullara/mustache.java — use instead when you want strictly logic-less templates portable across languages.
- jknack/handlebars.java — use instead when you want Mustache semantics plus helpers and want to share templates with a JavaScript frontend.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.x | ~1999–2001 | Original SourceForge project (Benjamin Geer et al.). |
| 2.0 | 2002 | Rewrite; FTL as broadly known today. |
| 2.3.0 | 2004 | Start of the long-lived 2.3 line still in use. |
| 2.3.19 | 2012 | Maintained by Dékány/Szegedi pre-Apache. |
| — | 2015-07 | Enters the Apache Incubator[^2]. |
| 2.3.24 | 2016 | Output formats / auto-escaping introduced[^4]. |
| — | 2018-03 | Graduates to Apache top-level project[^2]. |
| 2.3.31 | 2021 | Continued 2.3.x maintenance releases. |
| 2.3.34 | 2024 | Recent maintenance release on the 2.3 line[^3]. |

## References

[^1]: Apache FreeMarker README and manual — project scope, MVC View usage, Java SE 8 minimum. https://freemarker.apache.org/docs/
[^2]: Apache FreeMarker incubation and graduation. https://freemarker.apache.org/ and https://incubator.apache.org/projects/freemarker.html
[^3]: FreeMarker version history / change log (stable releases). https://freemarker.apache.org/docs/app_versions.html
[^4]: FreeMarker Manual — auto-escaping and output formats. https://freemarker.apache.org/docs/dgui_misc_autoescaping.html

## Tags

java, template-engine, ftl, server-side-rendering, html-generation, code-generation, apache, jvm, mvc, backward-compatibility, ssti-risk
