# pylint-dev/pylint

> A static analyser for Python that infers values rather than trusting types — thorough, opinionated, and knowingly slow.

[GitHub repo](https://github.com/pylint-dev/pylint) ·
[Documentation](https://pylint.readthedocs.io/en/latest/) ·
[License: GPL-2.0](https://github.com/pylint-dev/pylint/blob/main/LICENSE)

## Overview

Pylint is one of the oldest Python linters still in active use, originally written at Logilab in the early 2000s[^1] and now maintained under the community `pylint-dev` organisation (formerly `PyCQA`, a rename the OpenSSF badges still trail). It analyses code without executing it: parsing each module, building an abstract representation via its sibling library `astroid`, and running a large battery of checks for errors, style violations, code smells, and refactoring opportunities.

The defining design choice — and the source of both its value and its reputation — is inference. Where most linters walk the raw AST and match syntactic patterns, pylint tries to infer the actual values that names resolve to. If your code does `import logging as argparse`, pylint still knows `argparse.error(...)` is a logging call[^2]. This catches a class of real bugs that pattern-matchers miss, at the cost of being substantially slower and occasionally wrong in ways that produce false positives. The project is candid about this tradeoff; the README quotes a user calling inference "the killer feature that keeps us using pylint despite how painfully slow it is."

Pylint is aimed at teams that want a coding standard enforced thoroughly and are willing to configure their way out of the noise. It is highly configurable, ships opinionated checks that are disabled by default, and supports a plugin ecosystem for framework-specific rules (Django, Pydantic, and others). It is GPL-2.0 licensed — copyleft, which is worth noting for anyone embedding or redistributing it as part of a proprietary toolchain.

## Getting Started

```bash
pip install pylint
# optional spell-checking (requires the enchant C library):
pip install "pylint[spelling]"
```

```bash
# lint a package or module
pylint mypackage/

# adoption on a legacy codebase: start with errors only, then widen
pylint --errors-only mypackage/
pylint --disable=C,R mypackage/     # silence convention + refactor first
```

Configuration lives in `pylintrc`, `.pylintrc`, or a `[tool.pylint]` table in `pyproject.toml`:

```toml
[tool.pylint.main]
jobs = 0                      # 0 = use all CPU cores
load-plugins = ["pylint.extensions.docparams"]

[tool.pylint."messages control"]
disable = ["missing-docstring", "too-few-public-methods"]
```

Each run prints per-message output and a score out of 10, computed from the ratio of messages to statements — a heuristic, not a rigorous metric, but a common CI gate.

## Architecture / How It Works

Pylint is best understood as two projects. `astroid` is the inference engine: it parses source into an enriched syntax tree whose nodes can be asked "what could this actually be?" and it performs the value/type inference that distinguishes pylint. Pylint proper is the checker framework layered on top.

Messages are grouped into categories, each with a letter prefix: **C**onvention, **R**efactor, **W**arning, **E**rror, **F**atal, and **I**nfo. Every check emits symbolic message IDs (e.g. `unused-import`, `no-member`) which are what you enable, disable, and reference in `# pylint: disable=...` inline comments. The stable symbolic names — as opposed to numeric codes alone — are a deliberate usability choice.

Checkers register against node types and are invoked as pylint walks the astroid tree. Because inference needs surrounding context (imports, class hierarchies, resolved names), pylint effectively reasons about a module as a whole rather than line-by-line, which is why it cannot be as trivially parallelised or incrementalised as a pure-AST linter. The `-j`/`jobs` option parallelises across files, not within a file.

The project also ships two standalone tools built on the same foundation: `pyreverse`, which generates UML package and class diagrams, and `symilar`, a duplicate-code detector that is also exposed as an in-linter check.

## Production Notes

**Speed is the dominant operational concern.** Inference is expensive, and pylint is routinely an order of magnitude slower than AST-only linters on the same tree. On large monorepos a full run can take minutes. Mitigations: run with `jobs = 0` to use all cores, lint only changed files in pre-commit hooks, and reserve the full-tree run for CI rather than the edit loop.

**False positives are a maintenance tax.** Inference occasionally can't resolve dynamic constructs (metaclasses, monkey-patching, C extensions without stubs), producing `no-member` and `import-error` noise. The standard remedies are targeted inline `disable` comments, `generated-members` / `ignored-modules` config, or dedicated plugins for the offending framework. Teams that adopt pylint without curating a config tend to abandon it under the noise.

**Adopt incrementally.** On a legacy codebase, enabling everything at once produces thousands of C and R messages that bury the E-category bugs that matter. The maintained advice is `--errors-only` first, then `--disable=C,R`, then re-enable categories as priorities allow.

**Version upgrades move the baseline.** New releases add checks and tighten inference, so a green build can go red purely from upgrading pylint — pinning the version in CI is standard practice. Pylint and astroid are released in lockstep; mismatched versions are a common cause of crashes and should be upgraded together. The current line supports Python 3.10 and above.

**Not an autofixer.** Pylint reports; it does not rewrite. Formatting and fixes come from other tools (black, isort, ruff, autoflake). Pylint is one stage in a pipeline, not the whole pipeline.

## When to Use / When Not

**Use when:**
- You want the deepest available bug-detection for Python and can absorb the runtime cost.
- Your code is partially or untyped, where inference finds issues a type-checker or AST linter would miss.
- You want a single tool covering errors, style, refactoring hints, and duplicate detection, with per-project configuration.
- You need framework-aware checks available via the plugin ecosystem.

**Avoid when:**
- Fast feedback in the edit loop is the priority — ruff is dramatically faster for the overlapping checks.
- You want auto-fixing; pylint only reports.
- GPL-2.0 is incompatible with how you need to redistribute your toolchain.
- Your codebase is heavily dynamic in ways inference handles poorly, and you don't want to invest in config tuning.

## Alternatives

- astral-sh/ruff — Rust-based linter and formatter, far faster, with auto-fix and a growing subset of pylint's checks; the common choice for the fast inner loop, often run alongside pylint rather than replacing it.
- PyCQA/flake8 — lightweight AST-based framework (pyflakes + pycodestyle + plugins); use when you want pep8/pyflakes coverage without inference overhead.
- python/mypy — static type checker; use instead when the goal is type correctness rather than lint/style, and often run in addition to pylint.
- microsoft/pyright — fast type checker powering Pylance; use for typing feedback in editors.
- PyCQA/bandit — security-focused static analysis; complementary, not a substitute for general linting.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | ~2003 | Initial release at Logilab, built on the logilab-astng inference library (later astroid)[^1]. |
| 1.0 | 2013-08 | First 1.0; supported both Python 2 and 3. |
| 2.0 | 2018-07 | Python 3 only; Python 2 support dropped[^3]. |
| 3.0 | 2023-10 | astroid 3.0, dropped older Python versions, removed long-deprecated options[^4]. |

*(Minor-version release dates above are approximate; consult the changelog for exact points.)*

## References

[^1]: Pylint origins at Logilab and the astroid/logilab-astng lineage. https://pylint.readthedocs.io/en/latest/
[^2]: "What differentiates Pylint?" — inference of actual values via astroid. Project README. https://github.com/pylint-dev/pylint#what-differentiates-pylint
[^3]: Pylint changelog / release notes (2.0 series). https://pylint.readthedocs.io/en/latest/whatsnew/index.html
[^4]: Pylint 3.0 "What's New". https://pylint.readthedocs.io/en/latest/whatsnew/3/3.0/index.html

## Tags

python, static-analysis, linter, code-quality, astroid, type-inference, pep8, cli, developer-tools, gpl
