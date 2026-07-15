# jendrikseipp/vulture

> Static analyzer that finds unused ("dead") Python code by name, trading precision for zero-runtime, zero-config speed.

[GitHub repo](https://github.com/jendrikseipp/vulture) ·
[License: MIT](https://github.com/jendrikseipp/vulture/blob/main/LICENSE)

## Overview

Vulture scans Python source with the standard library `ast` module, records
every defined name and every used name, and reports the definitions that were
never referenced: unused functions, classes, methods, attributes, variables,
imports, and unreachable code. It is maintained by Jendrik Seipp and has been
in development since 2017[^1]. At ~4.7k stars it is the de facto tool for the
narrow job of "tell me what code is dead," a task neither type checkers nor
linters do well.

The defining tradeoff is stated plainly in its own README: because Python is
dynamic, a purely static analyzer "is likely to miss some dead code" and, worse,
will flag code that is only reachable through reflection, plugins, framework
callbacks, or serialization. Vulture's analysis "ignores scopes and only takes
object names into account"[^2] — so a method used anywhere is considered used
everywhere with that name, and a name defined but only reached via `getattr`
looks dead. This is a deliberate design: full reachability analysis is
undecidable in Python, so Vulture picks a cheap, fast, name-based heuristic and
hands you tools (confidence values, whitelists) to manage the resulting false
positives.

It complements rather than replaces `pyflakes`: it shares pyflakes' output
syntax but reaches beyond unused imports and locals into dead top-level
functions, classes, and attributes across a whole package.

## Getting Started

```bash
pip install vulture        # or: conda install -c conda-forge vulture
```

```bash
vulture myscript.py mypackage/          # scan files and/or directories
vulture mypackage/ --min-confidence 80  # suppress the noisiest guesses
```

```python
# vulture also runs as a library
import vulture
v = vulture.Vulture()
v.scavenge(["."])
for item in v.get_unused_code():
    print(item.name, item.first_lineno, item.confidence)
```

After deleting reported code, run Vulture again — removing one dead symbol
often exposes the next.

## Architecture / How It Works

Vulture builds an AST for each `*.py` file and walks it, populating two sets of
names: definitions (via `visit_*` handlers for `FunctionDef`, `ClassDef`,
assignments, imports, attributes, etc.) and usages (name loads, attribute
access, decorator references). Anything defined but never used is emitted.

The crucial simplification is that this is **name-based, not binding-based**.
Vulture does not resolve which object a name refers to; it treats the program as
a bag of identifier strings. That is why using `obj.greet` once marks every
`greet` method as live, and why a class only instantiated via a string passed to
`getattr` reads as dead. It is fast and language-version-tolerant precisely
because it skips the hard semantic work a type checker would do.

Each finding carries a **confidence value** from 60% to 100%[^2]:

| Code type | Confidence |
|-----------|------------|
| function/method/class arguments, unreachable code | 100% |
| imports | 90% |
| attributes, classes, functions, methods, properties, variables | 60% |

100% items are structurally certain (an argument the function never reads, code
after a `return`). The 60% items are "very rough estimates" — this is where the
tool guesses. Unreachable-code detection is a separate pass: it flags statements
after `return`/`break`/`continue`/`raise` and unsatisfiable `if`/`while`
conditions.

False positives are managed through **whitelists**: `--make-whitelist` emits a
Python file that simulates usage of every reported name; you commit it and pass
it back on the next run so those names count as used. Vulture ships curated
whitelists for common libraries under `vulture/whitelists/`. Configuration lives
in `pyproject.toml` under `[tool.vulture]` (CLI flags with dashes replaced by
underscores).

## Production Notes

- **Scope-blindness cuts both ways.** Because matching is by name alone, Vulture
  both under-reports (any shared method name hides a truly-dead one) and
  over-reports (framework entry points look dead). Treat 60%-confidence output as
  a review queue, not a delete list. Many teams run `--min-confidence 80` or
  `100` in CI and reserve lower thresholds for manual audits.
- **Dynamic code is the perennial footgun.** Flask/Django views, Celery tasks,
  pytest fixtures, `__getattr__`/`getattr` dispatch, plugin registries, and
  Pydantic/ORM fields routinely trip it. Mitigations: `--ignore-decorators
  "@app.route"`, `--ignore-names visit_*,do_*`, per-project whitelists, or the
  shipped framework whitelists. Names beginning with `_` are ignored by design.
- **Whitelists are maintenance debt.** A regenerated whitelist can silently
  re-mask code that genuinely became dead later; whitelists need periodic
  pruning or they defeat the tool's purpose. The `del`-keyword and `_name`
  conventions are lighter-weight for the common "unused argument" case.
- **Exit code 3 means dead code found** (not an error) — distinct from 1 (bad
  input) and 2 (bad arguments). CI scripts that treat any non-zero exit as
  failure will work, but you lose the ability to distinguish "dirty" from
  "broke."
- **flake8 interop is partial.** Vulture honors `# noqa: F401` (unused import)
  and `# noqa: F841` (unused variable) but the maintainer recommends whitelists
  over `noqa`, and its own `V1xx` error codes remain undocumented/hidden in the
  README pending a decision on `noqa` support.
- **No incremental mode.** Vulture re-parses the whole path set each run;
  it is fast per-file but does full scans, so very large monorepos pay for it in
  pre-commit hooks. Scope the `paths` config accordingly.

## When to Use / When Not

**Use when:**
- You want to find genuinely-dead functions/classes/attributes across a package,
  not just unused imports.
- You can run it over library **and** test suite together to surface untested,
  unreferenced code.
- You want a fast, dependency-light check with no execution or instrumentation.

**Avoid / augment when:**
- Your codebase is heavily dynamic (plugins, reflection, DI, web routes) — the
  false-positive rate may exceed the signal without heavy whitelisting.
- You need certainty about reachability: `coverage` run over a real test suite
  finds dead code far more reliably (at the cost of needing all branches
  exercised).
- You want dead-code detection folded into one linter pass — a modern all-in-one
  like Ruff may already cover your unused-import/variable needs.

## Alternatives

- PyCQA/pyflakes — use instead when you only need unused imports and unused
  locals as part of general error checking; narrower but near-zero false
  positives.
- nedbat/coveragepy (coverage.py) — use instead when you have a thorough test
  suite and want reachability proven by execution rather than guessed statically.
- astral-sh/ruff — use instead when you want unused-import/variable rules built
  into a single fast linter, rather than a dedicated dead-code pass.
- asottile/dead — use instead when you want a smaller, git-aware AST checker with
  fewer knobs.
- albertas/deadcode — use instead when you want an alternative unused-code finder
  with autofix/removal built in.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2017 | First 1.x line; AST name-based dead-code detection[^1]. |
| 2.0 | 2020-08-11 | 2.x series; Python 3-only, dropped legacy support[^3]. |
| 2.3 | 2021-01-16 | Pinned as the pre-commit example revision in the README[^3]. |
| 2.13 | 2024-10-02 | Ongoing 2.x maintenance releases[^3]. |
| 2.16 | 2026-03-25 | Latest tagged release[^3]. |

Development is steady rather than fast-moving: a mature 2.x line with
incremental releases, no announced 3.0, and an actively triaged issue tracker.

## References

[^1]: Repository metadata, `jendrikseipp/vulture` — created 2017-03-06, MIT license, ~4.7k stars / ~195 forks, last push 2026-04-30. https://github.com/jendrikseipp/vulture
[^2]: Vulture README — analysis method, confidence-value table, whitelists, `pyproject.toml` config, exit codes. https://github.com/jendrikseipp/vulture#readme
[^3]: Vulture release tags (GitHub Releases API) — v2.0 (2020-08-11), v2.3 (2021-01-16), v2.13 (2024-10-02), v2.16 (2026-03-25). https://github.com/jendrikseipp/vulture/releases

## Tags

python, static-analysis, dead-code, linter, ast, code-quality, cli, unused-code, refactoring, developer-tools
