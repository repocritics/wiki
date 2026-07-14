# pallets/jinja

> The template engine that most of Python's server-side rendering, config generation, and DevOps tooling quietly runs on.

[GitHub repo](https://github.com/pallets/jinja) ·
[Official website](https://jinja.palletsprojects.com) ·
[License: BSD-3-Clause](https://github.com/pallets/jinja/blob/main/LICENSE.txt)

## Overview

Jinja is a text templating engine for Python: templates are strings with
`{{ ... }}` expression markers and `{% ... %}` statement tags, rendered against a
data context to produce a final document. It was created by Armin Ronacher and
first released as "Jinja2" in 2008[^1], and is now maintained by the Pallets
organization alongside Flask, Werkzeug, Click, and MarkupSafe. The PyPI package
is still named `Jinja2`; the project and repository dropped the "2" after the 3.0
rewrite line[^2].

Its reach is much wider than web HTML. Jinja is Flask's default template engine,
but it is also the templating layer inside Ansible, Salt, Sphinx, Pelican, dbt,
and countless config-generation and code-generation pipelines. For a large share
of its users the output is not HTML at all — it is YAML, nginx configs, SQL, or
shell scripts. This matters because Jinja's HTML-focused safety feature
(autoescaping) is off by default and irrelevant to those users, while its real
cross-cutting risk (template injection) applies to all of them.

The defining tension is Ronacher's stated philosophy: application logic belongs
in Python, but the engine "shouldn't make the template designer's job difficult
by restricting functionality too much." Jinja is deliberately more expressive
than logic-less engines like Mustache — it has loops, conditionals, macros,
inheritance, and arbitrary Python method calls on context objects. That
expressiveness is the reason it renders Ansible playbooks well, and also the
reason server-side template injection (SSTI) is a recurring, high-severity class
of vulnerability around it.

## Getting Started

```bash
pip install Jinja2
```

```python
from jinja2 import Environment, FileSystemLoader, select_autoescape

env = Environment(
    loader=FileSystemLoader("templates"),
    autoescape=select_autoescape(),   # OFF unless you ask — enable for HTML
)

template = env.get_template("page.html")
print(template.render(users=[{"username": "ada", "url": "/u/ada"}]))
```

```jinja
{# templates/page.html — inheritance + a loop #}
{% extends "base.html" %}
{% block content %}
  <ul>
  {% for user in users %}
    <li><a href="{{ user.url }}">{{ user.username }}</a></li>
  {% endfor %}
  </ul>
{% endblock %}
```

## Architecture / How It Works

Jinja does not interpret templates directly. On first load it lexes and parses a
template into an AST, then **compiles that AST into Python source code**, which is
`exec`'d into a module-level `root` generator function. Rendering is then just
calling generated Python. This is why Jinja is fast relative to naive interpreters
and why tracebacks can be made to point at the original template line rather than
generated code — the compiler rewrites line numbers[^3].

Key pieces:

- **`Environment`** — the central config object holding loaders, filters, tests,
  globals, autoescape policy, and delimiter settings. Templates are cached on the
  Environment after first compile.
- **Loaders** — `FileSystemLoader`, `PackageLoader`, `DictLoader`, `ChoiceLoader`,
  etc. Template inheritance (`{% extends %}`) and inclusion (`{% include %}`) go
  through the loader.
- **`MarkupSafe`** — a separate Pallets C/Python library providing the `Markup`
  string type and `escape()`. Autoescaping is implemented here, not in Jinja
  itself; escaped output is tracked at the string-type level so already-safe
  values are not double-escaped.
- **`SandboxedEnvironment`** — a subclass that intercepts attribute access and
  operations to block reaching "unsafe" Python internals (`__class__`,
  `__globals__`, etc.). Intended for semi-trusted template authors.
- **Bytecode cache** — `FileSystemBytecodeCache` / `MemcachedBytecodeCache`
  persist the compiled bytecode so process restarts skip recompilation.
- **Async** — `Environment(enable_async=True)` generates coroutine-based render
  code so templates can `await` async functions and iterate async generators.

Undefined handling is pluggable: the default `Undefined` renders missing
variables as empty and is lenient; `StrictUndefined` raises on any undefined
access; `ChainableUndefined` allows deep attribute chains. The default's leniency
is a frequent source of silently-blank output.

## Production Notes

**Server-side template injection is the headline risk.** Never build a template
from user input (`Template(user_string)` or `render_template_string(user_input)`).
Because Jinja executes real Python at render time, a payload like `{{7*7}}`
confirms injection and further payloads walk the object graph to reach
`os.system`. This is a standard, well-tooled attack (tplmap, PayloadsAllTheThings)
and the single most common Jinja CVE pattern in downstream apps.

**The sandbox is a mitigation, not a boundary.** `SandboxedEnvironment` raises the
bar for untrusted template *authors*, but it has a history of escape CVEs
(e.g. format-string and `str.format` based breakouts, and `attr`/method access
gaps)[^4]. For genuinely hostile input, treat the sandbox as defense-in-depth, not
a guarantee — do not render attacker-controlled templates and rely on the sandbox
alone.

**Autoescape defaults bite people.** A bare `Environment()` does *not* escape
output. Flask configures autoescaping for `.html`/`.xml` templates, which trains
developers to assume it is always on; it is not when you construct your own
Environment or render a non-HTML extension. Pass `autoescape=select_autoescape()`
or `autoescape=True` explicitly for any HTML you emit.

**Undefined leniency hides bugs.** In production, `StrictUndefined` is usually the
right choice — the default renders a mistyped `{{ user.emial }}` as an empty
string and ships silently. Ansible famously layers its own undefined behavior on
top, which is a common source of "why is this variable blank" confusion.

**Performance / caching.** Compilation is the expensive step; rendering is cheap.
Long-running processes benefit from the on-Environment template cache automatically.
For short-lived processes (CLI tools, serverless), configure a
`FileSystemBytecodeCache` so compilation is not repeated per invocation. Enabling
`enable_async` adds overhead to every render — only turn it on where you actually
await.

**Upgrade friction.** Jinja 3.0 (2021) dropped Python 2 and 3.5 and moved the
codebase to type-hinted Python 3[^2]. Jinja 3.1 removed several long-deprecated
APIs (some `Markup`/`Environment` internals) and raised the minimum Python version;
projects pinning very old third-party filters occasionally break on the 3.1 line.
The 3.1.x series has shipped multiple security patch releases — pin a floor, not an
exact version, and track Pallets security advisories.

## When to Use / When Not

**Use when:**
- You render HTML or text from Python and want inheritance, macros, and filters.
- You are in the Flask/Pallets ecosystem, or a tool (Ansible, dbt, Sphinx) that
  already standardizes on Jinja.
- You generate config/code and want expressive templates with trusted authors.
- You need i18n (via Babel) or async rendering in an existing Python stack.

**Avoid when:**
- You must render fully untrusted, attacker-authored templates — no amount of
  sandboxing makes this safe; use a logic-less engine or a data format instead.
- You want logic-less, portable templates shared across languages — Mustache/
  Handlebars fit that constraint better.
- You are not in Python — use a native port rather than shelling out.
- You need compile-time-checked templates — Jinja errors are runtime.

## Alternatives

- sqlalchemy/mako — faster in some workloads, embeds real Python inline; less safe
  by design. Use when you want maximum templating power for trusted authors.
- mitsuhiko/minijinja — the original author's Rust reimplementation with Python
  bindings; use when you need Jinja-compatible syntax in Rust or want a lighter
  dependency.
- django/django — the Django Template Language is intentionally more restrictive;
  use when you are already in Django and want its auto-escaping-by-default model.
- Mustache / Handlebars (e.g. defunkt/mustache) — logic-less and cross-language;
  use when templates are shared across stacks or authored by untrusted users.
- Chameleon (malthe/chameleon) — attribute-based (TAL) HTML templating; use when
  you want valid-XML templates that render in a browser as-is.

## History

| Version | Date | Notes |
|---------|------|-------|
| 2.0 (Jinja2) | 2008 | First public release by Armin Ronacher; AST-to-Python compiler[^1]. |
| 2.x line | 2008–2020 | Sandbox, autoescaping via MarkupSafe, async support, Babel i18n. |
| 3.0 | 2021-05-11 | Python 3.6+ only; dropped Python 2/3.5; full type hints[^2]. |
| 3.1 | 2022-03-24 | Removed deprecated APIs; raised minimum Python; security hardening. |
| 3.1.x | 2022–2025 | Ongoing patch line with multiple CVE fixes (xmlattr, sandbox)[^4]. |

## References

[^1]: Jinja documentation, project introduction and history. https://jinja.palletsprojects.com/
[^2]: Pallets, "Jinja 3.0" changelog and release notes. https://jinja.palletsprojects.com/en/stable/changes/
[^3]: Jinja docs, "API — compilation and the Environment." https://jinja.palletsprojects.com/en/stable/api/
[^4]: Pallets Jinja security advisories (GitHub Security Advisories). https://github.com/pallets/jinja/security/advisories

## Tags

python, template-engine, html, jinja, flask, pallets, ssti, rendering, code-generation, sandboxing
