# astral-sh/ruff

> An extremely fast Python linter and code formatter, written in Rust — a single binary that replaces Flake8, isort, Black, pydocstyle, pyupgrade, and dozens of plugins.

[GitHub repo](https://github.com/astral-sh/ruff) ·
[Official website](https://docs.astral.sh/ruff) ·
[License: MIT](https://github.com/astral-sh/ruff/blob/main/LICENSE)

## Overview

Ruff is a Python linter and formatter implemented in Rust, first released in 2022 by Charlie Marsh and now developed by Astral, the company behind the `uv` package manager and the `ty` type checker[^1]. Its pitch is consolidation and speed: rather than composing Flake8 with a stack of plugins, plus isort, plus Black, plus pyupgrade — each a separate process re-parsing your files — Ruff re-implements over 900 lint rules and a Black-compatible formatter as one native binary that walks the source once. The commonly cited figure is 10–100x faster than the tools it replaces[^2]; in practice this turns whole-repo linting from tens of seconds into a fraction of a second, which is what makes it viable as an editor-on-save and pre-commit tool.

The defining tension is **breadth versus fidelity**. Ruff does not wrap the original tools; it ports their behavior into Rust. That port is close but not always byte-identical, so a small number of rules and formatter edge cases diverge from the upstream they emulate, and Ruff tracks this with parity notes and issue labels rather than claiming perfect equivalence[^3]. For the vast majority of codebases the difference is invisible; for teams with deeply customized Flake8 plugin configs or unusual style expectations, migration means re-verifying that the ported rules match intent.

Ruff is a linter and formatter, not a type checker. Type checking is a separate Astral product (`ty`, still pre-1.0 as of 2026); Ruff's static analysis is syntactic and flow-based, not a full type system. It is used across large projects including FastAPI, pandas, SciPy, Apache Airflow, and Hugging Face Transformers[^2].

## Getting Started

```shell
# Run without installing, via uv:
uvx ruff check    # Lint the current directory.
uvx ruff format   # Format the current directory.

# Or install:
uv tool install ruff@latest   # Global tool.
uv add --dev ruff             # Project dev dependency.
pip install ruff              # Or plain pip.
```

Minimal `pyproject.toml` configuration:

```toml
[tool.ruff]
line-length = 88
target-version = "py310"

[tool.ruff.lint]
# Default is ["E4", "E7", "E9", "F"]; add more rule families as needed.
select = ["E", "F", "I", "B", "UP"]   # pycodestyle, Pyflakes, isort, bugbear, pyupgrade
ignore = []

[tool.ruff.format]
quote-style = "double"
```

```shell
ruff check --fix        # Lint and auto-apply safe fixes (e.g. remove unused imports).
ruff format             # Apply the formatter.
ruff check --select I --fix   # Sort imports only.
```

As a pre-commit hook via `ruff-pre-commit`:

```yaml
- repo: https://github.com/astral-sh/ruff-pre-commit
  rev: v0.15.21
  hooks:
    - id: ruff-check
      args: [--fix]
    - id: ruff-format
```

## Architecture / How It Works

Ruff is a Cargo workspace of many crates. The core pipeline is: tokenize and parse Python source into an AST (a hand-written Rust parser, `ruff_python_parser`, that replaced an earlier dependency on RustPython's parser), build semantic context, then run rules against the AST and token stream in a single traversal per file.

- **Parser.** Ruff maintains its own error-tolerant Python parser targeting current CPython grammar (Python 3.14 support is tracked in the README). Error tolerance matters because a linter must produce useful diagnostics on code that does not fully parse.
- **Rules as first-party ports.** Every rule — whether it originated in Flake8, isort, pyupgrade, pydocstyle, or pandas-vet — is re-implemented in Rust rather than shelled out to. Rules are grouped into families addressed by code prefix (`F` = Pyflakes, `E`/`W` = pycodestyle, `I` = isort, `B` = flake8-bugbear, `UP` = pyupgrade, `RUF` = Ruff-native, and many more).
- **Fixes.** Many rules carry autofixes. Ruff classifies fixes as *safe* or *unsafe*; `--fix` applies only safe fixes by default, and `--unsafe-fixes` opts into the rest. Unsafe fixes are ones that may change runtime behavior or drop comments.
- **Formatter.** A separate subsystem, built on a fork of Rome's `rome_formatter` intermediate-representation approach, aiming for drop-in parity with Black's style[^4]. It is a distinct code path from the linter, invoked as `ruff format`.
- **Import resolution** draws on the algorithm from Microsoft's Pyright[^4], used for isort-style import sorting and module-awareness.
- **Configuration discovery** is hierarchical: Ruff walks up from each file to find the nearest `pyproject.toml` / `ruff.toml` / `.ruff.toml`, and configs cascade — the monorepo story that lets subdirectories override root settings.
- **Caching.** Results are cached (in `.ruff_cache`) keyed on file content and settings, so unchanged files are not re-analyzed on the next run.

The linter and formatter are deliberately decoupled: the default lint rule set omits stylistic `E` codes that would fight a formatter, on the assumption you run `ruff format` (or Black) for whitespace concerns.

## Production Notes

**Versioning is pre-1.0 and the version *is* the contract.** Ruff is still on 0.x. Because linters gate CI, the project pins behavioral changes to minor releases: new rules, changed rule behavior, and rule stabilizations land in minor bumps, and unstable work is quarantined behind `preview = true`[^5]. The practical rule is to **pin an exact Ruff version** in CI and pre-commit (`rev:`), because a floating version can add a rule that fails a previously-green build.

**Preview mode changes defaults.** Enabling `preview` does not just unlock new rules — it also expands the default rule set (adding `B`, `UP`, `RUF` families and more) and can change formatter output. Treat `preview = true` as an unstable channel, not a feature flag you leave on in production without review.

**Safe vs unsafe fixes.** `--fix` is conservative by design, but even "safe" fixes can surprise you if a rule's autofix removes code you intended to keep (e.g. an unused import that a linter cannot see is imported for a side effect). Review autofix diffs; do not run `--fix` blind in CI against an unreviewed tree.

**Parity gaps with the tools it replaces.** Migrating off Flake8-plus-plugins is mostly mechanical, but a handful of rules differ from upstream, and Ruff intentionally does not implement some plugins (or implements a stricter/looser variant). If your team relied on a specific plugin's exact behavior, verify the ported rule rather than assuming equivalence. Formatter output is Black-*compatible*, not Black-*identical* in every edge case — expect a one-time reformat diff when switching.

**Noqa and rule selection are a real migration cost.** Existing `# noqa: E501` comments keep working, but the large default and preview rule sets mean adopting Ruff on a legacy codebase produces a wall of new diagnostics. The common path is to start from the small default (`F` plus a subset of `E`), then widen `select` incrementally, using `--add-noqa` to baseline existing violations.

**It is not a type checker.** Ruff will not catch type errors; pair it with mypy, Pyright, or Astral's own `ty`. Rules that look type-aware (e.g. some flake8-bugbear or flake8-type-checking checks) are heuristic and AST-based, not backed by inference.

## When to Use / When Not

**Use when:**
- You want one fast tool to replace a Flake8 + isort + Black + pyupgrade stack.
- Linting speed matters — large repos, lint-on-save in the editor, or pre-commit hooks where seconds of latency hurt adoption.
- You want autofixes and import sorting integrated with linting rather than as separate passes.
- You're standardizing tooling across a monorepo and want cascading per-directory config.

**Avoid / think twice when:**
- You depend on a niche Flake8 plugin Ruff hasn't ported, or on exact upstream behavior you cannot re-verify.
- You need type checking — that's a different tool.
- You require a 1.0-stable, rarely-changing tool with a long deprecation runway; Ruff moves fast and is still 0.x, so unpinned upgrades can change CI outcomes.
- Your style requirements diverge from Black — the formatter is opinionated and Black-shaped, with limited knobs.

## Alternatives

- PyCQA/flake8 — the tool Ruff most directly replaces; use it if you need a specific plugin Ruff hasn't ported or want the mature, slow-moving reference behavior.
- psf/black — the formatter Ruff emulates; use it directly if you want the canonical implementation rather than a compatible port.
- PyCQA/isort — standalone import sorter; Ruff's `I` rules cover most of it, but isort has more configuration surface for unusual layouts.
- microsoft/pyright / python/mypy — type checkers; complementary, not competitors — run one alongside Ruff.
- astral-sh/ty — Astral's own Rust type checker (pre-1.0); pairs naturally with Ruff but is not yet production-stable.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial release | 2022-08 | First public release by Charlie Marsh[^1]. |
| Astral founded | 2023-2024 | Company formed around Ruff and uv[^1]. |
| 0.1.0 | 2023-10 | First 0.1 milestone; API/config stability commitments begin. |
| 0.4.0 | 2024-04 | Rewritten Rust parser (replacing the RustPython-derived one), faster and more error-tolerant. |
| 0.5.0 | 2024-06 | Standalone installers (`astral.sh/ruff/install.sh`) added. |
| 0.6–0.9 | 2024-2025 | Formatter maturation, expanded rules, preview-mode iteration. |
| 0.15.x | 2026 | Current line as of writing (0.15.21); Python 3.14 support, ongoing preview default expansion[^5]. |

## References

[^1]: Astral, "Announcing Astral: The Company Behind Ruff." https://astral.sh/blog/announcing-astral-the-company-behind-ruff
[^2]: Ruff README and documentation, astral-sh/ruff. https://docs.astral.sh/ruff/
[^3]: Ruff FAQ, "How does Ruff's linter compare to Flake8?" https://docs.astral.sh/ruff/faq/
[^4]: Ruff README, Acknowledgements (Rome `rome_formatter` fork; Pyright import resolver). https://github.com/astral-sh/ruff
[^5]: Ruff docs, "Preview mode" and versioning. https://docs.astral.sh/ruff/preview/

## Tags

python, linter, formatter, static-analysis, rust, code-quality, developer-tools, cli, flake8, black, pre-commit
