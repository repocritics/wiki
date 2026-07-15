# PyCQA/flake8

> A launcher that runs pyflakes, pycodestyle, and mccabe under one command and one plugin system — the long-standing default Python linter, now being displaced by Ruff.

[GitHub repo](https://github.com/PyCQA/flake8) ·
[Official website](https://flake8.pycqa.org) ·
[License: MIT](https://github.com/PyCQA/flake8/blob/main/LICENSE)

## Overview

Flake8 is not itself a linter. It is a thin orchestration layer that runs three separate tools — PyFlakes (logical error detection), pycodestyle (PEP 8 style checks), and Ned Batchelder's McCabe script (cyclomatic complexity) — and merges their output into a single per-file report[^1]. It was created by Tarek Ziadé and is currently maintained by Anthony Sottile and Ian Cordasco under the PyCQA (Python Code Quality Authority) umbrella. GitHub reports the license as unrecognized ("NOASSERTION") because of a non-standard header, but the LICENSE file is verbatim MIT.

Its defining design choice is the plugin architecture: since the 3.0 rewrite (2016), any linter that emits checks in the expected format can register itself through the `flake8.extension` setuptools entry point and run inside the same pass[^2]. This is why the ecosystem around flake8 (flake8-bugbear, flake8-comprehensions, pep8-naming, flake8-docstrings, and hundreds more) exists — flake8 is the socket those plugins plug into. Each check has a code prefix: `E`/`W` from pycodestyle, `F` from pyflakes, `C` from mccabe, and per-plugin prefixes (`B` for bugbear, `N` for pep8-naming, and so on).

The defining tension of the project in 2026 is that Ruff — a Rust reimplementation of flake8 and most of its popular plugins — runs one to two orders of magnitude faster and is a single dependency-free binary. Flake8 is stable, ubiquitous in existing CI, and still the reference for many plugin authors, but new projects increasingly reach for Ruff first.

## Getting Started

```bash
python -m pip install flake8
```

Run it against a path (or `.` for the whole tree):

```bash
flake8 path/to/code/
```

A minimal `.flake8` config file:

```ini
[flake8]
max-line-length = 100
extend-ignore = E203, W503
exclude = .git,__pycache__,build,dist
max-complexity = 10
```

Inline suppression uses `# noqa` (silence a whole line) or `# noqa: E501` (silence specific codes, comma-separated). A `# flake8: noqa` line skips the entire file[^1]. The `noqa` token is case-insensitive, but the colon before a code list is required — `# noqa E501` (no colon) silences *all* codes on the line, a common surprise.

## Architecture / How It Works

Flake8's job is coordination, not analysis. The pipeline is:

1. **Config discovery** — walks up from the target files looking for `setup.cfg`, `tox.ini`, or `.flake8`, and merges with command-line options.
2. **Plugin loading** — enumerates installed `flake8.extension` and `flake8.formatting` entry points via `importlib.metadata`. Each plugin advertises the code prefix(es) it owns.
3. **File processing** — a `FileProcessor` reads each file once, builds the token stream and the AST, and hands the appropriate representation to each check. Logical-line checks (pycodestyle-style) get tokens; AST checks (pyflakes, most plugins) get the parsed tree. Building the AST once and sharing it is the main efficiency argument for running plugins inside flake8 rather than as separate processes.
4. **Parallelism** — by default flake8 forks a process pool (via `multiprocessing`) and shards files across workers, then merges and sorts results.
5. **noqa filtering** — a regex pass over each file's raw lines removes reported violations that fall on a suppressed line.
6. **Formatting** — the default formatter prints `path:row:col: CODE message`; alternative formatters (e.g. `--format=json`-style plugins) register through `flake8.formatting`.

Because flake8 only glues, its checking behavior is exactly the union of its constituent tools' behaviors. It performs no cross-file analysis, no type inference, and no import resolution beyond what pyflakes does syntactically. That is deliberate: flake8 is fast and predictable precisely because it never leaves the single-file AST.

Flake8 pins its three core dependencies to narrow version ranges (a specific `pycodestyle`, `pyflakes`, and `mccabe` band per flake8 release). This is the source of most installation conflicts — see below.

## Production Notes

**No `pyproject.toml` support.** Flake8 reads only INI-style config (`setup.cfg`, `tox.ini`, `.flake8`). It does not and, per a long-standing maintainer position, will not read `[tool.flake8]` from `pyproject.toml`[^3]. Projects that consolidate every other tool's config into `pyproject.toml` must keep a separate flake8 file. The `Flake8-pyproject` third-party plugin exists as a workaround. This single friction point is one of the most-cited reasons teams migrate to Ruff, which does read `pyproject.toml`.

**Dependency pinning conflicts.** Because flake8 constrains `pycodestyle`/`pyflakes`/`mccabe` to tight ranges, installing a newer standalone `pycodestyle` alongside flake8 — or two tools that each pin different flake8-compatible ranges — frequently produces `pip` resolver conflicts. Pin flake8 itself and let it drag in the matching versions; do not pin its dependencies independently.

**Performance.** Flake8 is fine on small and medium repos but scales poorly on large monorepos: startup imports every installed plugin, and the per-file Python AST walk is inherently slower than a compiled linter. On a large codebase the difference versus Ruff is measured in seconds-to-minutes versus milliseconds. If flake8 runtime is a CI bottleneck, that is the migration signal.

**`noqa` drift.** Blanket `# noqa` (no code) is a frequent code-smell in older codebases — it silences everything on the line, hiding real regressions. `flake8-noqa` and the `--disable-noqa` flag help audit this; Ruff and newer setups prefer requiring explicit codes.

**Plugin version coupling.** A flake8 major bump occasionally breaks plugins that relied on internal APIs (the plugin loading and `Checker` internals are not a stability contract). Pin plugin versions in the same lockfile as flake8 and upgrade them together.

**Not a formatter.** Flake8 reports style violations but does not fix them. Pair it with Black (formatting) and isort/autoflake (import cleanup); disable the flake8 checks that conflict with Black — commonly `E203` and `W503` — via `extend-ignore`.

## When to Use / When Not

**Use when:**
- You have an existing flake8 setup with a curated plugin list and no performance pain — there is no urgency to migrate.
- You depend on a specific plugin that Ruff has not reimplemented.
- You want the exact, well-understood behavior of pyflakes + pycodestyle rather than a re-implementation.
- Your team already knows the `E`/`W`/`F` codes and the `# noqa` workflow.

**Avoid / reconsider when:**
- You're starting a new project in 2026 — evaluate Ruff first; it covers most of flake8's rules plus formatting in one fast tool.
- Lint time is a CI bottleneck on a large codebase.
- You want a single `pyproject.toml` for all tooling with no extra config file.
- You need auto-fixing, type-aware checks, or cross-file analysis — flake8 does none of these.

## Alternatives

- astral-sh/ruff — Rust reimplementation of flake8 + many plugins + a formatter; use when you want the same checks 10–100× faster, `pyproject.toml` config, and auto-fix in one binary. The default recommendation for new projects.
- PyCQA/pylint — deeper, semantic, cross-reference analysis with far more checks; use when you want thoroughness and can tolerate slower runs and more opinionated/false-positive-prone output.
- PyCQA/pycodestyle — the style checker alone; use when you want only PEP 8 checks without the flake8 launcher or plugins.
- PyCQA/pyflakes — the logical-error checker alone; use when you want unused-import/undefined-name detection with zero style noise.
- psf/black — a formatter, not a linter; complementary to flake8 rather than a replacement, and pairs with it in most setups.

## History

| Version | Date | Notes |
|---------|------|-------|
| 2.x | 2013–2016 | Original launcher; ad-hoc plugin handling. |
| 3.0 | 2016-05 | Complete rewrite: entry-point plugin architecture, `FileProcessor`, parallel processing[^2]. |
| 3.6 | 2018-06 | Structured `# noqa: CODE` handling improvements. |
| 4.0 | 2021-10 | Dropped older Python versions; internal cleanups[^4]. |
| 5.0 | 2022-07 | Moved to `importlib.metadata` for plugin discovery. |
| 6.0 | 2022-11 | Dropped Python 3.6; removed some deprecated options[^4]. |
| 7.0 | 2024-01 | Dropped Python 3.8; current major line[^4]. |

## References

[^1]: Flake8 README and documentation — feature set, `noqa` semantics, wrapped tools. https://flake8.pycqa.org/en/latest/
[^2]: Flake8 plugin development guide — `flake8.extension` / `flake8.formatting` entry points, processor model. https://flake8.pycqa.org/en/latest/plugin-development/index.html
[^3]: Flake8 issue thread on `pyproject.toml` configuration support (maintainer declines). https://github.com/PyCQA/flake8/issues/234
[^4]: Flake8 release notes / changelog. https://flake8.pycqa.org/en/latest/release-notes/index.html
[^5]: Ruff — Rust linter reimplementing flake8 and plugins. https://github.com/astral-sh/ruff

## Tags

python, linter, static-analysis, code-quality, pep8, style-checker, pyflakes, pycodestyle, cli, plugin-architecture, pycqa
