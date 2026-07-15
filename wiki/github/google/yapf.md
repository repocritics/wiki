# google/yapf

> A configurable Python code formatter that reflows entire files to a minimal-penalty layout, using an algorithm ported from clang-format.

[GitHub repo](https://github.com/google/yapf) ·
No official website ·
[License: Apache-2.0](https://github.com/google/yapf/blob/main/LICENSE)

## Overview

YAPF is a whole-file Python formatter that Google open-sourced in 2015[^1]. Its stated goal is to produce code "as good as the code that a programmer would write if they were following the style guide" — not merely to fix style violations. Unlike `autopep8`, which patches only PEP 8 infractions and leaves the rest of your layout untouched, YAPF discards the existing line-breaking and recomputes it from scratch. The algorithm is a direct descendant of clang-format's (originally by Daniel Jasper): it enumerates possible split points, assigns each a penalty, and searches for the arrangement that minimizes total penalty under a column limit[^2].

The defining tension is **configurability versus consensus**. YAPF exposes 40-plus "knobs" (`COLUMN_LIMIT`, `DEDENT_CLOSING_BRACKETS`, `SPLIT_BEFORE_LOGICAL_OPERATOR`, `ARITHMETIC_PRECEDENCE_INDICATION`, and many more) plus four base styles — `pep8` (default), `google`, `yapf`, and `facebook`. This flexibility was its main selling point in 2015-2018 and is now its main liability: the community largely consolidated around `psf/black`, whose entire pitch is the absence of options ("you get one style, stop arguing"). YAPF's remaining niche is teams that genuinely need a house style Black won't produce, and codebases (Google's own, historically) built around a specific configuration.

YAPF is still versioned in the 0.x range and its README describes it as "beta"; released versions change formatting behavior between minor releases[^1]. It is explicitly "not an official Google product" — code that happens to be owned by Google. At ~14k stars it is widely known but no longer the default choice for new projects.

## Getting Started

```bash
pip install yapf
```

Format in place, recursively, using the Google base style with two overrides:

```bash
yapf -i -r --style='{based_on_style: google, column_limit: 100, indent_width: 2}' src/
```

As a library:

```python
from yapf.yapflib.yapf_api import FormatCode

formatted, changed = FormatCode("f ( a = 1, b = 2 )", style_config="pep8")
# formatted == "f(a=1, b=2)\n"
# changed   == True
```

Persist a style by writing a `.style.yapf`, `setup.cfg` `[yapf]`, or `pyproject.toml` `[tool.yapf]` section. `--style-help` prints every knob with its current value, which you can redirect into a config file.

## Architecture / How It Works

YAPF runs four conceptual stages:

1. **Parse** — the source is tokenized and parsed into a concrete syntax tree using a lib2to3-derived grammar (vendored into the project). This choice is the root of YAPF's slow adoption of new syntax: as of the current README, PEP 695 type-parameter syntax (3.12) and PEP 701 f-strings (3.12) are still unsupported and tracked as open issues[^3].
2. **Build logical lines** — the tree is flattened into "unwrapped lines" (logical statements), annotated with every legal split point and a per-point split penalty.
3. **Solve** — a search over the space of split decisions finds the layout minimizing total penalty subject to `COLUMN_LIMIT`. This is the clang-format inheritance: formatting is posed as an optimization problem, not a set of rewrite rules. It is why YAPF can find non-obvious "nicest" arrangements — and why it is comparatively slow.
4. **Emit** — the chosen layout is rendered back to text; `FormatCode` / `FormatFile` return `(text, changed)` or `(text, encoding, changed)`.

The knob system is not cosmetic: many knobs directly change penalties or hard constraints in stages 2-3, so two configurations can produce structurally different trees, not just different whitespace. `based_on_style` behaves like subclassing — you inherit a base style's knob values and override individual keys.

Style resolution is hierarchical and searches upward through parent directories: command line, then `.style.yapf`, then `setup.cfg`, then `pyproject.toml`, then `~/.config/yapf/style`, falling back to PEP 8. Exclusions come from `-e` patterns, a `.yapfignore` file, or `[tool.yapfignore] ignore_patterns` in `pyproject.toml`.

## Production Notes

**Formatter drift is the main CI hazard.** Because output changes between minor versions, an unpinned YAPF will silently reformat a codebase the day a new release lands, producing enormous no-op diffs and failing `--diff` gate checks. Pin an exact version in your lockfile / pre-commit config and bump it deliberately. The `-d`/`--diff` flag returns non-zero when changes are needed, which is the standard CI assertion that code is already formatted.

**Behavioral changes hide inside knobs.** The `DISABLE_ENDING_COMMA_HEURISTIC` semantics changed in v0.40.3, and a separate `DISABLE_SPLIT_LIST_WITH_COMMENT` flag was split out — to preserve old behavior you must now set both[^4]. Upgrades that "shouldn't" reflow code sometimes do because a heuristic was refactored. Read the CHANGELOG before bumping.

**Performance.** The penalty-solving approach is meaningfully slower than Black and dramatically slower than Rust-based formatters. Use `-p`/`--parallel` when formatting many files. For very large monorepos this cost is real; ruff format exists in part because Python formatters were a build-time bottleneck.

**Scope limits.** YAPF only lays out code — it does not sort imports (pair it with `PyCQA/isort`), does not remove unused imports, and does not upgrade syntax. It reflows but does not rewrite semantics.

**Gradual adoption.** `-l START-END` formats a line range, and the bundled `yapf-diff` reformats only the lines touched by a unified diff (`git diff -U0 | yapf-diff -i`). This lets a large legacy codebase converge file-by-file instead of in one churning commit. This is one area where YAPF's ranged formatting is genuinely more capable than Black's whole-file model.

**Idempotency.** YAPF aims to be idempotent (formatting twice equals formatting once) but historically has had bugs where a second pass changed output under certain knob combinations. If you script it into a hook, assert stability on your codebase after any version bump.

## When to Use / When Not

**Use when:**
- You need a specific house style Black cannot express (custom column limit behavior, bracket dedent/indent rules, precedence-aware operator spacing).
- You are working in Google-style codebases already standardized on a YAPF config.
- You want per-range formatting to migrate a legacy codebase incrementally via `yapf-diff`.

**Avoid when:**
- You are starting fresh and want to end style debates — Black's zero-config opinion is the path of least resistance and has the larger ecosystem.
- Formatting speed matters at scale — ruff format is orders of magnitude faster.
- You need current-syntax coverage — PEP 695 and PEP 701 f-string internals are not yet supported.
- You want import sorting or lint-plus-format in one tool — YAPF does neither.

## Alternatives

- psf/black — near-zero-config, opinionated formatter; use instead when you want one canonical style and no bikeshedding. This is what most of the ecosystem migrated to.
- astral-sh/ruff — Rust linter that also ships a Black-compatible formatter; use when speed and a unified lint+format toolchain matter more than configurability.
- google/pyink — Google's fork of Black with a few style deviations; use when you want Black's engine but Google's minor formatting preferences.
- hhatto/autopep8 — patches only PEP 8 violations without reflowing everything; use when you want a light-touch fixer rather than a whole-file reformatter.
- PyCQA/isort — import sorter, complementary rather than competing; pair with YAPF since YAPF never touches import order.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-03 | Open-sourced by Google; clang-format-derived algorithm[^1]. |
| 0.x | 2015-2026 | Long-lived beta; formatting behavior evolves across minor releases. |
| 0.40.3 | 2023 | `DISABLE_ENDING_COMMA_HEURISTIC` behavior changed; `DISABLE_SPLIT_LIST_WITH_COMMENT` added[^4]. |

Repository created 2015-03-18; still actively maintained, with the most recent push in mid-2026[^5]. YAPF has never shipped a 1.0.

## References

[^1]: YAPF README — Introduction, Installation, and Formatting style sections. https://github.com/google/yapf/blob/main/README.md
[^2]: clang-format, the C++ formatter whose algorithm YAPF ports (Daniel Jasper). https://clang.llvm.org/docs/ClangFormat.html
[^3]: YAPF README — "Python features not yet supported": PEP 695 (issue #1170), PEP 701 (issue #1136). https://github.com/google/yapf/issues/1170
[^4]: YAPF README — `DISABLE_ENDING_COMMA_HEURISTIC` note referencing the v0.40.3 change and CHANGELOG. https://github.com/google/yapf/blob/main/CHANGELOG.md
[^5]: GitHub API repository metadata for google/yapf (stars, license, timestamps), retrieved 2026-07. https://api.github.com/repos/google/yapf

## Tags

python, formatter, code-formatter, code-style, pep8, clang-format, developer-tools, cli, google, autoformatter
