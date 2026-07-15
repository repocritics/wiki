# twigphp/Twig

> The compiled, auto-escaping template language for PHP — maintained alongside Symfony and used well beyond it.

[GitHub repo](https://github.com/twigphp/Twig) ·
[Official website](https://twig.symfony.com/) ·
[License: BSD-3-Clause](https://github.com/twigphp/Twig/blob/3.x/LICENSE)

## Overview

Twig is a template language and engine for PHP, created by Fabien Potencier (founder of the Symfony framework) in 2009[^1]. Its syntax was borrowed from Python's Django and Jinja template languages: `{{ ... }}` prints an expression, `{% ... %}` runs a tag, `{# ... #}` is a comment. The deliberate design constraint is that templates are *not* PHP — the language exposes a restricted set of filters, functions, and control structures rather than letting arbitrary PHP run in the view layer. That restriction is the whole point: designers and templates cannot reach into application internals, and output is escaped by default.

The defining implementation choice is that Twig **compiles templates to plain PHP classes** rather than interpreting them. A template is lexed, parsed to a node tree, and emitted as a cached PHP class whose `doDisplay()` method writes the output. After the first compile, rendering is an ordinary method call, so runtime cost is close to hand-written PHP — the tradeoff is a compile step and a writable cache directory that has to be managed. This is what lets a "slow, safe, designer-friendly" DSL stay fast in production.

Twig long ago outgrew Symfony. Drupal adopted it as its theming engine in Drupal 8 (2015), and it is the default view layer across the Symfony ecosystem and many standalone PHP projects. It remains actively maintained under the Symfony organization, with 8.3k+ stars and commits landing regularly as of mid-2026.

## Getting Started

```bash
composer require twig/twig
```

```php
<?php
require __DIR__ . '/vendor/autoload.php';

use Twig\Environment;
use Twig\Loader\FilesystemLoader;

$loader = new FilesystemLoader(__DIR__ . '/templates');
$twig = new Environment($loader, [
    'cache' => __DIR__ . '/var/cache/twig',  // compiled PHP goes here
    'auto_reload' => true,                    // recompile on source change (dev)
]);

echo $twig->render('index.html.twig', ['name' => 'Tom', 'items' => $items]);
```

```twig
{# templates/index.html.twig #}
{% extends 'base.html.twig' %}

{% block body %}
  <h1>Hello {{ name }}</h1>       {# auto HTML-escaped #}
  <ul>
    {% for item in items %}
      <li>{{ item.title|upper }}</li>
    {% endfor %}
  </ul>
{% endblock %}
```

## Architecture / How It Works

The central object is the **`Environment`**, wired to one or more **loaders** (`FilesystemLoader`, `ArrayLoader`, `ChainLoader`) that resolve a template name to source. Compilation is a fixed pipeline:

1. **Lexer** — source string → token stream.
2. **Parser** — tokens → a node tree (AST). Extensions can register `NodeVisitor`s that rewrite the tree here.
3. **Compiler** — the node tree emits PHP source for a class extending Twig's `Template`, written to the cache directory.
4. **Render** — the compiled class is `include`d and its `doDisplay()` executed. Subsequent requests skip straight to step 4 if the cache is warm.

**Auto-escaping** is applied at compile time: with the default `html` strategy, printed expressions are wrapped in an escape call unless explicitly marked safe. The escaping is context-sensitive by strategy (`html`, `js`, `css`, `url`, `html_attr`), which matters because HTML-escaping a value interpolated into a `<script>` or a URL is not safe.

**Extensions** are the extension point for everything: filters (`|upper`), functions (`path()`), tests (`is defined`), operators, tags, and node visitors all register through an extension. `CoreExtension` ships the built-ins; framework integrations add their own.

**Inheritance and reuse** come in several forms that are easy to conflate: `extends` + `block` (vertical inheritance), `use` (horizontal reuse of blocks), `include` (render another template), `embed` (include + override blocks), and `macro` (reusable callable snippets). Macros deliberately do **not** inherit the caller's variable context — you pass arguments explicitly.

**The sandbox** (`SandboxExtension`) restricts which tags, filters, methods, and properties a template may touch. It is the mechanism intended for rendering untrusted, user-supplied templates.

## Production Notes

**Server-Side Template Injection (SSTI) is the headline security risk.** Rendering a template whose *source* comes from user input — not just its variables — without the sandbox lets an attacker reach object methods and, in practice, achieve code execution. Twig SSTI is a well-documented attack class. Rule: never `createTemplate()` from user input outside a locked-down `SandboxExtension`; passing user data as *variables* into a fixed template is safe.

**`|raw` and escaping mistakes are the XSS surface.** Auto-escaping only protects the default path. Any `{{ x|raw }}`, a wrong escaping strategy for the context, or building HTML by string concatenation reopens XSS. Audit every `|raw`.

**`strict_variables` is off by default.** Undefined variables and missing keys silently render as empty/null, so template typos fail quietly. Turn on `strict_variables` (at least in dev/CI) to surface them.

**Cache management is an operational task, not an afterthought.** The `cache` directory must be writable and should be warmed at deploy time; a cold cache compiles on first hit. Leave `auto_reload` **on in dev** (recompile when source changes) and **off in prod** (skip the mtime check) for best throughput, and pair it with PHP OPcache so compiled classes are not re-parsed each request. Stale caches after a deploy are a common "why is my change not showing" bug — clear the compiled cache on release.

**Keep logic out of templates.** The language intentionally omits arbitrary PHP; heavy loops, data shaping, and business logic belong in the controller/presenter, not the view. Fighting this by cramming filters and inline expressions produces templates that are slow to compile and hard to test.

**Whitespace control is a footgun.** `{%- ... -%}` and the `spaceless`/`{% apply spaces %}` behavior trip people up in whitespace-sensitive output (email, YAML, code generation).

**Upgrade pain is concentrated at the majors.** Twig 2.0 renamed every class from the `Twig_Environment` underscore convention to real namespaces (`Twig\Environment`) — a mechanical but repo-wide break. Twig 2.x and 3.x were developed in parallel with matching features, 3.x differing mainly by dropping deprecated APIs and raising the PHP floor, so the 2→3 jump is mostly "resolve the deprecation warnings you were already getting." Run the deprecation-collecting mode before a major bump.

## When to Use / When Not

**Use when:**
- You're on Symfony or Drupal — Twig is the native, integrated view layer.
- You want output that is auto-escaped by default and a template layer that can't reach into app internals.
- You want near-native render speed from a designer-friendly DSL (the compile-to-PHP payoff).
- You need to render templates authored by semi-trusted users and can lock them behind the sandbox.

**Avoid when:**
- You're already in Laravel — Blade is the tightly integrated default and mixing engines adds friction.
- You'd rather write plain PHP views and skip learning a DSL — a native-PHP templating library fits better.
- Your output is a tiny script or API with no real view layer — the compile step and cache dir are overhead you don't need.
- You want logic-less, cross-language templates shared with non-PHP stacks — Mustache-style engines travel better.

## Alternatives

- smarty-php/smarty — use when you have legacy Smarty templates or want a long-established engine outside the Symfony orbit.
- laravel/framework — use when you're in Laravel; Blade is the native, compile-to-PHP view layer there.
- thephpleague/plates — use when you'd rather write plain PHP templates and avoid a separate DSL entirely.
- nette/latte — use when you want compiled templates with context-aware escaping and are open to the Nette ecosystem.
- bobthecow/mustache.php — use when you want logic-less, language-agnostic templates shared across stacks.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2009-10 | Project created by Fabien Potencier; syntax modeled on Django/Jinja[^1]. |
| 1.0 | 2011 | First stable line; `Twig_`-prefixed classes, PHP 5. |
| 2.0 | 2017-05 | Namespaced classes (`Twig\...`), PHP 7 required[^2]. |
| 3.0 | 2019-11 | Feature-parallel with 2.x minus deprecated APIs; PHP 7.2.5+ required[^3]. |

## References

[^1]: Twig documentation — introduction and origins. https://twig.symfony.com/doc/3.x/intro.html
[^2]: Twig changelog (2.x line). https://github.com/twigphp/Twig/blob/3.x/CHANGELOG
[^3]: Twig documentation, "Twig 3 vs Twig 2" — installation and requirements. https://twig.symfony.com/doc/3.x/installation.html

## Tags

php, template-engine, templating, twig, symfony, auto-escaping, compiled-templates, view-layer, drupal, ssti
