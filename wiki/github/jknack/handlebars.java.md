# jknack/handlebars.java

> A Java port of Handlebars/Mustache — logic-less templates with a helper API for the parts logic-less can't reach.

[GitHub repo](https://github.com/jknack/handlebars.java) ·
[Official website](http://jknack.github.io/handlebars.java) ·
[License: Apache-2.0](https://github.com/jknack/handlebars.java/blob/master/LICENSE)

## Overview

Handlebars.java is a JVM implementation of the Handlebars templating language,
itself a superset of Mustache[^1]. It was written by Edgar Espina and has been
maintained under the `jknack` account since 2012[^2]. Templates are "logic-less"
by design — the syntax expresses variables, sections, and partials, not
arbitrary code — and the escape hatch for real logic is a Java-side helper API
rather than expressions embedded in the template.

The library's defining tension is exactly that logic-less premise. Mustache
compatibility keeps templates portable and safe from casual code injection, but
real applications need conditionals, formatting, and iteration with behavior. The
answer is helpers: `if`, `each`, `with`, `unless` ship built in, while string and
conditional helpers (`eq`, `gt`, `capitalize`, `dateFormat`) exist but must be
registered explicitly — a frequent first-day surprise. The result sits between a
strict Mustache engine and a full expression language like Thymeleaf or FreeMarker.

Handlebars.java is a mature, slow-moving dependency. It is widely embedded in
JVM tooling (notably Swagger/OpenAPI code generators and various web frameworks)
where its Mustache compatibility lets it consume the same `.mustache` templates
other ecosystems use. As of 2026 it carries roughly 1,600 stars and ~390 forks —
modest numbers that undersell its transitive reach through those generators.

## Getting Started

```xml
<dependency>
  <groupId>com.github.jknack</groupId>
  <artifactId>handlebars</artifactId>
  <version>4.5.3</version>
</dependency>
```

```java
Handlebars handlebars = new Handlebars();

// Inline compilation — no template loader needed
Template template = handlebars.compileInline("Hello {{this}}!");
System.out.println(template.apply("Handlebars.java"));  // Hello Handlebars.java!

// Or load "greeting.hbs" from the classpath
Template t2 = handlebars.compile("greeting");
System.out.println(t2.apply(Map.of("name", "Edgar")));
```

## Architecture / How It Works

The pipeline is: a `TemplateLoader` resolves a name to a `TemplateSource`, the
parser compiles that source into a `Template`, and `template.apply(context)`
renders against a `Context`. Compilation is eager and, per the README, very fast;
the compiled `Template` is thread-safe and meant to be reused.

**Template loaders.** `ClassPathTemplateLoader` (default), `FileTemplateLoader`,
and `SpringTemplateLoader` (in the `handlebars-springmvc` module) each carry a
configurable `prefix` and `suffix` (default `.hbs`). Loader choice is where most
"template not found" problems originate.

**Value resolvers.** How `{{name}}` maps onto a Java object is pluggable.
`JavaBeanValueResolver` (getters), `FieldValueResolver` (fields),
`MapValueResolver`, `MethodValueResolver`, and `JsonNodeValueResolver` (Jackson,
in the `handlebars-jackson2` module) can be composed on a `Context`. The default
stack uses JavaBean + Map + Method resolution, which means it invokes public
methods reflectively — relevant to the security notes below.

**Helpers.** A helper is either an implementation of the `Helper<T>` interface or
any public method on a registered "helper source" class, resolved by name. Since
1.1.0, helpers can also be authored in JavaScript and executed through a bundled
JS engine[^3]. Block helpers receive an `Options` object exposing positional
params, a hash map (`class="..."`), and `options.fn()`/`options.inverse()` for
the body and else-block.

**Template inheritance.** The `block`/`partial` helper pair implements layout
inheritance — a `{{#block "title"}}` in a parent defines a named region that a
child `{{#partial "title"}}` overrides[^4]. This is Handlebars.java's own
extension, not part of the Mustache spec.

**Caching.** By default there is no cache — every `apply` re-parses. Production
setups pick `ConcurrentMapTemplateCache` or `HighConcurrencyTemplateCache`, both
of which watch source files and reload on change. A Guava-backed cache exists as
a separate module. The cache interface has no `put`: all work happens inside
`get`, keyed on `TemplateSource`.

## Production Notes

**Server-Side Template Injection is the headline risk.** Because the default
resolver stack invokes methods reflectively, a template string built from
untrusted input can reach arbitrary Java classes and achieve remote code
execution — Handlebars.java is a well-documented SSTI target in the security
literature[^5]. Templates must be treated as trusted code. Never
`compileInline` user-supplied strings, and never let user input choose a template
name without an allowlist.

**Registration surprises.** Only a small set of helpers is active by default.
`StringHelpers` and `ConditionalHelpers` (including `eq`, `and`, `or`, `not`)
require an explicit `registerHelpers(...)` call. Code copied from tutorials that
assumes `{{#if (eq a b)}}` works out of the box will fail with an "unknown
helper" error until conditional helpers are registered.

**Caching is opt-in and easy to forget.** The default no-cache behavior re-parses
on every request, which is fine in development and a measurable hot-path cost in
production. Wire a `ConcurrentMapTemplateCache` explicitly; do not assume the
engine memoizes.

**Precompilation ships an old handlebars.js.** The `precompile` helper emits
JavaScript using a bundled `handlebars-v1.3.0.js` by default (overridable to
2.0.0). If your front end runs a modern handlebars.js runtime, the precompiled
template format can mismatch — pin versions on both sides.

**Runtime requirements moved.** Handlebars 4.4+ requires Java 17; the 4.3.x line
(the last to run on Java 8) is explicitly marked "NOT MAINTAINED" in the
README[^2]. Staying on Java 8 means staying on an unmaintained branch.

**Error reporting is a genuine strength.** Syntax and partial-resolution errors
report `file:line:column` with a caret pointing at the offending token and a
partial call stack — better than most JVM template engines. Helper/runtime errors
are best-effort on location.

## When to Use / When Not

**Use when:**
- You need Mustache-compatible templates on the JVM and want to reuse `.mustache`
  files shared with JS or other language ecosystems.
- You want logic-less templates with a clean Java helper extension point.
- You need template inheritance (`block`/`partial`) or i18n via `ResourceBundle`.
- You're already consuming it transitively (Swagger/OpenAPI generators) and want
  the same engine for your own views.

**Avoid when:**
- Any template or template name can come from untrusted input — the SSTI surface
  is real and RCE-capable.
- You want rich in-template expressions and natural-templating HTML — Thymeleaf or
  FreeMarker fit better.
- You are pinned to Java 8 long-term — you'd be on the unmaintained 4.3.x line.
- You want a high-velocity, actively-featured project; this is stable maintenance,
  with releases arriving irregularly.

## Alternatives

- spullara/mustache.java — stricter, faster pure-Mustache engine with no helper
  extensions; use it when you want Mustache and nothing more.
- thymeleaf/thymeleaf — use when you want natural (valid-HTML) templates and rich
  expression logic tightly integrated with Spring.
- apache/freemarker — use when you need a full-featured template language with
  in-template logic and macros, injection surface accepted.
- jknack's own handlebars.js (handlebars-lang/handlebars.js) — use when the
  runtime is Node/browser rather than the JVM.
- pebbletemplates/pebble — use when you want Twig/Jinja-style syntax with
  autoescaping and inheritance on the JVM.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2012 | Initial release by Edgar Espina[^2]. |
| 1.1.0 | 2013 | JavaScript-authored helpers added[^3]. |
| 4.0.x | 2018 | 4.x line; modular artifacts (springmvc, jackson2, proto). |
| 4.2.0 | 2020-04-25 | Maintenance release on the 4.x line. |
| 4.3.0 | 2021-10-12 | Last line supporting Java 8 (later marked unmaintained). |
| 4.4.0 | 2024-03-07 | Java 17 required baseline. |
| 4.5.0 | 2025-08-07 | Current minor line. |
| 4.5.3 | 2026-06-30 | Latest release at time of writing. |

## References

[^1]: Handlebars language, of which the Java port is a superset of Mustache. https://handlebarsjs.com/
[^2]: Handlebars.java README — requirements, maintenance status, and usage. https://github.com/jknack/handlebars.java
[^3]: README, "With plain JavaScript" — helpers authored in JS since 1.1.0. https://github.com/jknack/handlebars.java#with-plain-javascript
[^4]: Template inheritance via block/partial. http://jknack.github.io/handlebars.java/reuse.html
[^5]: Server-Side Template Injection in Handlebars.java (PortSwigger / community research on template-engine RCE). https://portswigger.net/research/server-side-template-injection

## Tags

java, jvm, template-engine, handlebars, mustache, logic-less-templates, server-side-rendering, i18n, templating, ssti-risk
