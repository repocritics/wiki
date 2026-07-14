# psf/black

> The uncompromising Python code formatter — near-zero configuration by design, formatting decided by the tool, not by you.

[GitHub repo](https://github.com/psf/black) ·
[Official website](https://black.readthedocs.io/en/stable/) ·
[License: MIT](https://github.com/psf/black/blob/main/LICENSE)

## Overview

Black is an opinionated, deterministic formatter for Python source. It was created by Łukasz Langa (a CPython core developer) and first released publicly in 2018[^1]; it moved under the Python Software Foundation umbrella, hence the `psf/black` path. The premise is deliberately narrow: you cede control over layout minutiae, and in return every Blackened file looks the same regardless of who wrote it, which shrinks diffs and removes formatting from code review entirely.

The defining tension is the word "uncompromising." Black exposes almost no style knobs — line length, string quote preference (it normalizes to double quotes), and a magic trailing comma are about the extent of user-facing choices. This is a feature, not an oversight: the project's whole value proposition collapses if teams can bikeshed the config. The cost is that if you disagree with a specific decision (its handling of long call chains, the way it explodes collections onto multiple lines), there is usually no supported way to opt out, and "this looks wrong" is frequently answered with "intended behaviour, see the Stability Policy"[^2].

For most of its life Black carried a `beta` label and warned that output could change between releases. That ended with the 22.1.0 release in early 2022, which stabilized the style and adopted a calendar-based versioning scheme (`YY.M.patch`)[^3]. Since then, stylistic changes are gated behind a preview flag and a documented stability guarantee, so pinning a version is the normal way to keep formatting reproducible across a team and CI.

## Getting Started

```sh
pip install black
# Jupyter notebook support:
pip install "black[jupyter]"
```

```sh
# Format in place
black src/

# Check-only mode for CI (exit code 1 if anything would change)
black --check --diff src/
```

```toml
# pyproject.toml — the only supported config location
[tool.black]
line-length = 88          # the default; 88 is intentional, not 79
target-version = ["py310", "py311"]
extend-exclude = "migrations/"
```

Black requires Python 3.10+ to run (it can still target and format code written for older Python versions via `target-version`). Standalone PyInstaller executables are published on each GitHub release for environments without a Python interpreter.

## Architecture / How It Works

Black is a whole-file, AST-driven reformatter, not a line-patcher. The pipeline is: parse the source into a concrete syntax tree using a vendored fork of lib2to3's grammar (`blib2to3`), discard the original formatting almost entirely, then re-emit the tree through a line-generator that makes layout decisions from scratch. Because it does not read your existing formatting, running Black twice is idempotent and the input's whitespace has no influence on the output — this is what makes results project-independent.

The one heuristic that surprises people is line-splitting. Black tries to fit an expression on one line under the line-length budget; if it does not fit, it "explodes" — breaking collections and call arguments one-per-line. The **magic trailing comma** is the escape hatch: if you put a trailing comma in a collection, Black treats that as an instruction to keep it exploded even when it would otherwise fit on one line. This is the closest thing to manual control the formatter offers, and understanding it removes most "why did Black do that" confusion.

A notable safety property: after formatting, Black re-parses its own output and asserts the AST is equivalent to the input (modulo positions and docstring whitespace). If the two ASTs differ, it aborts rather than emit code that could change behavior. This check costs time; `--fast` skips it. The default 88-character line length was chosen empirically as roughly 10% longer than the traditional 79, which the author found reduced the number of forced line breaks meaningfully without hurting readability[^4].

Related tooling ships in-repo: `blackd` (an HTTP daemon for editor integrations), `black-primer`/the internal formatting test runner, and first-class integration as a `pre-commit` hook, which is the most common way teams enforce it.

## Production Notes

- **Version pinning is mandatory for reproducibility.** Because Black uses calendar versions and occasionally changes formatting in a new year's release, an unpinned `black` in CI can suddenly flag a clean codebase as needing reformatting. Pin the exact version in `requirements`/`pyproject.toml` and in your pre-commit `rev`, and bump deliberately.
- **The stable/preview split.** New style changes land behind `--preview` (and individual `--enable-unstable-feature` flags) for a release or two before graduating. Do not run `--preview` in CI unless you accept churn; use it to test what the next stable style will do to your diff.
- **It formats, it does not lint.** Black has no opinion on imports, unused variables, or naming. It is routinely paired with an import sorter and a linter. Older setups used `isort` plus `flake8`; many teams have since consolidated onto Ruff, which reimplements Black-compatible formatting and linting in one Rust binary and runs far faster.
- **Line-length interplay with linters.** If you also run `flake8`/`pycodestyle`, you must disable or reconfigure their E501 line-length and a handful of whitespace rules (E203, W503) because Black's output intentionally violates some `pycodestyle` defaults. This mismatch is a classic first-day-with-Black failure.
- **Large monorepo rollout.** Introducing Black to an existing large codebase produces one enormous diff. The recommended path is a single "reformat the world" commit that you add to `.git-blame-ignore-revs` so `git blame` skips it. Doing it piecemeal creates merge-conflict churn.
- **Speed.** For very large trees the AST-safety re-parse dominates runtime; `--fast` and parallelism help, but if raw throughput matters, Ruff's formatter is materially faster.

## When to Use / When Not

**Use when:**
- You want to end formatting debates on a team and make diffs minimal.
- You value determinism and reproducible CI checks over per-developer style preferences.
- You're standardizing a shared or open-source codebase where "looks the same everywhere" is worth more than any individual layout opinion.

**Avoid (or reconsider) when:**
- You need fine-grained control over layout — Black will not give it to you, by design.
- You're already adopting Ruff for linting and want one tool; Ruff's formatter is Black-compatible and may make a separate Black install redundant.
- Your codebase must support a Python older than 3.10 for the tool *runtime* (you can still format old syntax, but the tool itself needs 3.10+).

## Alternatives

- astral-sh/ruff — use instead when you want Black-compatible formatting plus linting and import sorting in a single fast Rust binary; increasingly the default choice for new projects.
- google/yapf — use when you actually want configurable style; YAPF is knob-heavy and can be tuned to a house style Black refuses to offer.
- hhatto/autopep8 — use when you only want minimal PEP 8 fixes applied to existing formatting rather than a full reflow.
- PyCQA/isort — complementary, not competing; pair it with Black for import ordering (or use Ruff's isort rules instead).
- pre-commit/pre-commit — the framework most teams use to run Black on commit; the enforcement layer around the formatter.

## History

| Version | Date | Notes |
|---------|------|-------|
| 18.3a0 | 2018-03 | First public release; beta-labelled, style not yet frozen[^1]. |
| 19.3b0 | 2019-03 | PyCon 2019 talk; broad adoption begins across major projects. |
| 20.8b0 | 2020-08 | Magic trailing comma, docstring normalization improvements. |
| 21.4b0 | 2021-04 | Python 3.9 pattern-matching support, jupyter extra. |
| 22.1.0 | 2022-01 | Style stabilized; calendar versioning; leaves beta[^3]. |
| 23.1.0 | 2023-01 | First "stable style" annual change (e.g. hug-parens tweaks). |
| 24.1.0 | 2024-01 | 2024 stable style; expanded preview features. |

## References

[^1]: Łukasz Langa, initial release of Black — 2018. https://github.com/psf/black/releases
[^2]: The Black Code Style: Stability Policy. https://black.readthedocs.io/en/stable/the_black_code_style/index.html#stability-policy
[^3]: "Black 22.1.0" release notes — the first stable, non-beta release. https://github.com/psf/black/releases/tag/22.1.0
[^4]: The Black Code Style: Current style (line length rationale). https://black.readthedocs.io/en/stable/the_black_code_style/current_style.html

## Tags

python, code-formatter, formatter, linter, developer-tools, pep8, ast, pre-commit-hook, cli
