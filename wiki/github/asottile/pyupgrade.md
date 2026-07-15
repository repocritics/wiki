# asottile/pyupgrade

> An AST-based tool (and pre-commit hook) that automatically rewrites Python source to newer-version syntax.

[GitHub repo](https://github.com/asottile/pyupgrade) ·
No official website ·
[License: MIT](https://github.com/asottile/pyupgrade/blob/main/LICENSE)

## Overview

pyupgrade rewrites Python source in place to use idioms available in newer
language versions: set/dict literals over `set([...])`, f-strings over
`.format()`, PEP 585/604 typing (`list[str]`, `int | None`) over
`typing.List` / `Optional`, `super()` over `super(C, self)`, removal of
now-redundant `from __future__` imports and `six`/`mock` compatibility
shims, and dozens more[^1]. It is written by Anthony Sottile, a CPython
contributor and long-time maintainer of the pre-commit framework, and is the
canonical member of a family of single-purpose asottile rewriters.

The defining design decision is that pyupgrade is **almost entirely
non-configurable**. There is no config file and only a handful of flags — a
target-version selector (`--py36-plus`, `--py311-plus`, `--py313-plus`, …)
plus a few opt-outs like `--keep-percent-format`, `--keep-mock`, and
`--keep-runtime-typing`. You do not choose which rules run; you choose which
Python version you support, and pyupgrade applies every rewrite that version
makes safe. This is deliberate: the tool encodes one person's opinion of
"the modern way to write this," and the lack of knobs is the product.

The consequence in 2026 is that pyupgrade is increasingly consumed *through*
other tools. Ruff reimplemented the bulk of pyupgrade's rewrites in Rust
under its `UP` rule prefix, so teams already running Ruff often drop the
standalone hook[^2]. pyupgrade remains actively maintained (releases through
v3.21.2, last pushed 2026), widely used as a pre-commit hook, and the
reference implementation that new rewrites still originate from — but its
role has shifted from "the tool you run" to "the rule set everyone copied."

## Getting Started

```bash
pip install pyupgrade
pyupgrade --py311-plus file1.py file2.py
```

As a pre-commit hook (the dominant usage), in `.pre-commit-config.yaml`:

```yaml
-   repo: https://github.com/asottile/pyupgrade
    rev: v3.21.2
    hooks:
    -   id: pyupgrade
        args: [--py311-plus]
```

pyupgrade edits files in place and exits non-zero when it changes anything —
the standard pre-commit contract, so a commit that triggers a rewrite fails
and must be re-staged. Always run it under version control and review the
diff; there is no dry-run-only mode that also reports without touching files.

## Architecture / How It Works

pyupgrade is a source-to-source rewriter, not a full compiler pass. Each file
goes through two layers:

1. **AST analysis.** The file is parsed with the standard-library `ast`
   module. Visitors walk the tree looking for rewritable patterns and record
   the token offsets where a change should happen. Because it uses the real
   Python parser, pyupgrade only runs on syntactically valid code for the
   interpreter executing it.
2. **Token rewriting.** Edits are applied over the token stream via
   `tokenize-rt` (another asottile library), not by unparsing the AST. This
   is what lets pyupgrade preserve the original formatting, comments, and
   whitespace of everything it does *not* touch — it surgically replaces spans
   rather than reprinting the file.

Internally the rewrites are organized as a plugin registry (`pyupgrade/_plugins/`),
each module owning one family of transforms (f-strings, typing PEPs, `six`
removal, `open` modes, etc.). This is an implementation detail, not a public
extension point — there is no supported plugin API for third parties.

Version targeting is central. Many rewrites are gated behind `--pyXX-plus`
because they are only valid at or above a given version: f-strings need 3.6+,
`X | Y` union syntax needs 3.10+ at runtime (or `from __future__ import
annotations` for annotation-only use). pyupgrade trusts your flag; it does not
detect your interpreter or read `python_requires`.

## Production Notes

**The target-version flag is a footgun.** pyupgrade applies every rewrite the
flag permits. Pass `--py311-plus` on a library that still supports 3.8 and it
will happily rewrite annotations to `int | None` and other 3.11-only forms,
producing code that raises `TypeError` at import time on your actual minimum
version. There is no safety net beyond you choosing the flag correctly; wire
it to the true floor of your support matrix, not the version on your laptop.

**Runtime-evaluated annotations break silently.** PEP 585/604 rewrites
(`List[str]` → `list[str]`, `Optional[x]` → `x | None`) only fire when the
file has `from __future__ import annotations`, *unless* you force them.
Frameworks that introspect annotations at runtime — pydantic, dataclasses
with certain configs, attrs, typer — can choke on the new-style forms on older
Pythons. Use `--keep-runtime-typing` for code whose annotations are evaluated,
not just type-checked.

**It only upgrades; it does not report or lint.** There is no "explain what
you would change" mode, no per-rule enable/disable, no severity levels. If you
want to adopt a subset of its behavior you cannot — it is all-or-nothing for
the chosen version. This is the single most common reason teams move to Ruff's
`UP` rules, where individual codes can be selected or ignored.

**Interaction with other formatters.** pyupgrade changes tokens (e.g.
`'%s' % x` → `'{}'.format(x)`, quote styles on `.encode()`), which can leave
lines that Black or Ruff-format then re-wrap. Order matters in pre-commit:
pyupgrade should generally run *before* the formatter so the formatter gets
the last word on layout. Running it after a formatter can produce churn.

**f-string conversion is intentionally timid.** pyupgrade will not build an
f-string if doing so makes the expression longer or the substitutions are
complex, to avoid hurting readability. Do not expect it to convert every
`.format()` call; the residue is by design, not a bug.

## When to Use / When Not

**Use when:**
- You run pre-commit and want zero-config, opinionated syntax modernization.
- You are migrating a codebase off `six` / Python 2 idioms or bumping a
  minimum-supported version and want the mechanical rewrites done for you.
- You want the reference behavior, tracked release-by-release, as the source
  of truth for "modern Python syntax."

**Avoid when:**
- You already run Ruff — enable the `UP` rules instead of adding a second
  hook that overlaps ~90% of them.
- You need to enable rewrites selectively or suppress specific ones;
  pyupgrade offers no such control.
- Your code depends on runtime introspection of annotations and you cannot
  audit the typing rewrites (use `--keep-runtime-typing` or skip it).

## Alternatives

- astral-sh/ruff — reimplements most pyupgrade rules under the `UP` prefix in Rust; use instead when you already run Ruff and want one fast tool with per-rule control.
- asottile/reorder_python_imports — sibling tool for import ordering and rewriting obsolete `six`/`__future__` imports; complementary, not overlapping.
- asottile/add-trailing-comma — sibling that manages trailing commas; often paired with pyupgrade in the same pre-commit config.
- adamchainz/django-upgrade — same in-place-rewrite pattern targeted at Django version idioms; use alongside pyupgrade for Django projects.
- PyCQA/autoflake — removes unused imports and variables; different scope, use when you want dead-code cleanup rather than syntax modernization.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-02 | First commit; Python 2→3 and early Python 3 syntax rewrites[^3]. |
| 2.x | ~2020 | Expanded typing / f-string rewrites; `tokenize-rt` rewrite engine matured. |
| 3.0 | ~2022 | Dropped Python 2 and oldest Python 3 support; plugin-based rewrite registry. |
| 3.21.2 | 2026 | Current release; PEP 696 / `--py313-plus` and later typing rewrites[^1]. |

## References

[^1]: pyupgrade README — implemented features and availability flags. https://github.com/asottile/pyupgrade#readme
[^2]: Ruff `pyupgrade` (UP) rule set. https://docs.astral.sh/ruff/rules/#pyupgrade-up
[^3]: pyupgrade repository, created 2017-02-28. https://github.com/asottile/pyupgrade

## Tags

python, linter, pre-commit, code-modernization, ast, source-rewriter, formatter, cli, developer-tools, typing
