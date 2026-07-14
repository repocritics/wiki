# python-poetry/poetry

> Python dependency management and packaging with a single `pyproject.toml`, a real resolver, and a committed lock file.

[GitHub repo](https://github.com/python-poetry/poetry) ·
[Official website](https://python-poetry.org) ·
[License: MIT](https://github.com/python-poetry/poetry/blob/main/LICENSE)

## Overview

Poetry is a dependency manager and build frontend for Python, created by Sébastien Eustace with a first commit in early 2018 and a 1.0 release in December 2019[^1]. It consolidates what used to be a scatter of files — `setup.py`, `requirements.txt`, `setup.cfg`, `MANIFEST.in`, `Pipfile` — into one `pyproject.toml`, and pins the resolved graph in a committed `poetry.lock`. Its target user is an application or library developer who wants reproducible installs and a coherent CLI (`poetry add`, `poetry install`, `poetry lock`, `poetry build`, `poetry publish`) instead of assembling pip, virtualenv, twine, and a hand-maintained requirements file.

Poetry's defining choice is a **full SAT-style dependency resolver** with a committed lock file, in contrast to pip's historically install-order-dependent behavior. This buys reproducibility but historically cost speed and produced hard-to-read conflict errors. The resolver is a PubGrub-derived algorithm (Poetry's `mixology`)[^2], and it will refuse to install rather than silently pick an inconsistent set.

The other defining tension is Poetry's long-standing use of a **non-standard metadata table**. For years Poetry declared project metadata under `[tool.poetry]` while the Python packaging ecosystem standardized on the PEP 621 `[project]` table[^3]. Poetry 2.0 (January 2025) finally added first-class support for `[project]`, but a large body of existing projects, tutorials, and CI still assume the older `[tool.poetry]` layout, so both live side by side in the wild[^4].

## Getting Started

Poetry should be installed in its own isolated environment, not into your project's virtualenv. Use the official installer or pipx:

```bash
curl -sSL https://install.python-poetry.org | python3 -
# or:  pipx install poetry
```

```bash
poetry new my-app          # scaffold a project
cd my-app
poetry add requests        # resolve, install, and write to pyproject.toml + poetry.lock
poetry install             # install the locked graph into a managed venv
poetry run pytest          # run a command inside that venv
```

A minimal PEP 621 `pyproject.toml` built by Poetry 2.x:

```toml
[project]
name = "my-app"
version = "0.1.0"
requires-python = ">=3.9"
dependencies = ["requests (>=2.28,<3.0)"]

[build-system]
requires = ["poetry-core>=2.0.0,<3.0.0"]
build-backend = "poetry.core.masonry.api"
```

## Architecture / How It Works

Poetry splits into two repositories. **poetry-core** is the dependency-free PEP 517 build backend (`poetry.core.masonry.api`) that turns a project into a wheel/sdist; it is what downstream tools invoke at build time and is deliberately kept small and stable. **poetry** (this repo) is the user-facing CLI, resolver, installer, and virtualenv manager, and it depends on a stack of the author's own libraries — `cleo` (CLI framework), `clikit`, `tomlkit` (style-preserving TOML), and `cachy`.

The resolution pipeline: Poetry reads declared constraints, queries package indexes (PyPI by default) for candidate versions, downloads and inspects metadata, and runs the PubGrub-style solver to find one consistent assignment for the whole graph. That assignment is serialized to `poetry.lock` with hashes. `poetry install` then reads the lock file — not `pyproject.toml` — so installs are deterministic across machines as long as the lock is committed.

Virtualenvs are managed by Poetry itself: by default it creates a venv per project in a central cache directory (configurable to `.venv` in-project via `virtualenvs.in-project`). Commands run through `poetry run` or `poetry shell` are executed inside that venv.

The plugin system (stabilized in 1.2) lets external packages add commands and hook into resolution. The most consequential use of it was **removing `poetry export` from core** and relocating it to `poetry-plugin-export`, which changed a workflow many CI pipelines relied on[^5].

Version constraints use a caret/tilde syntax layered over PEP 440 (`^1.2` means `>=1.2,<2.0`). Poetry historically also required an **upper bound on the Python version** itself for locking, which interacts badly with libraries that only set a lower bound — a frequent source of "SolverProblemError" reports.

## Production Notes

**Resolver speed and conflict messages.** On large graphs the solver has historically been slow, sometimes minutes, because it must download metadata (and occasionally full sdists) to learn transitive constraints. When it fails, the error names the conflicting packages but rarely the human-fixable root cause; debugging often means running with `-vvv` and reading the constraint chain by hand.

**The Python caret footgun.** Writing `python = "^3.9"` expands to `>=3.9,<4.0`, and Poetry propagates that upper bound into resolution. A dependency that declares `requires-python: >=3.9` with no upper bound can then produce a solver conflict against your bounded root. The common fix is `>=3.9,<4.0` written explicitly, or avoiding caret on `python`.

**`poetry export` is no longer built in.** Pipelines that did `poetry export -f requirements.txt` to feed pip or Docker layers must now install `poetry-plugin-export`, or (2.x) use `poetry export` only after adding the plugin. Silent CI breakage on upgrade is common here[^5].

**Lock file churn and format changes.** The `poetry.lock` format has changed across major versions; upgrading Poetry can rewrite the whole lock even with no dependency changes, producing large, noisy diffs. Pin the Poetry version in CI (e.g. via pipx or a fixed installer) so contributors don't regenerate incompatible locks.

**Don't `pip install poetry` into your project venv.** Doing so entangles Poetry's own dependency tree with your project's and causes resolution conflicts. Use the official installer or pipx to keep it isolated.

**Metadata migration.** Moving an existing `[tool.poetry]` project to the PEP 621 `[project]` table (2.x) is mostly mechanical but not automatic; dynamic fields, dependency groups, and the constraint syntax differences need manual attention. Many projects simply stay on `[tool.poetry]`, which remains supported.

## When to Use / When Not

**Use when:**
- You want reproducible installs with a committed lock file and hash pinning.
- You build and publish libraries or apps and want one tool for dependencies, building, and publishing.
- You value a strict resolver that refuses inconsistent graphs over a fast but permissive one.
- Your team is willing to standardize on one workflow and pin the Poetry version.

**Avoid when:**
- Install/resolve speed is critical and the graph is large — uv resolves the same graphs dramatically faster.
- You need the leanest possible standards-only setup — pip + a PEP 621 backend (hatchling, flit-core) has fewer moving parts.
- You are in a data-science stack that leans on conda/mamba for non-Python binary dependencies.
- You want to avoid a bespoke CLI and its own library stack in your toolchain.

## Alternatives

- astral-sh/uv — use instead when resolve/install speed matters; a Rust resolver + installer that also manages Python versions, now the fastest option and rapidly displacing Poetry for many teams.
- pypa/hatch — use instead when you want a standards-first (PEP 621) project + build + environment manager backed by PyPA.
- pdm-project/pdm — use instead when you want PEP 621 metadata and PEP 582/local-package workflows with a Poetry-like CLI.
- pypa/pipenv — use instead when you only need application-level `Pipfile` + lock and no build/publish story.
- pypa/pip — use instead when you want the standard installer directly with a requirements file and no resolver ceremony.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0.0 | 2019-12 | First stable release; `pyproject.toml` + `poetry.lock` workflow[^1]. |
| 1.1.0 | 2020-10 | New installer and parallel installs; resolver improvements. |
| 1.2.0 | 2022-08 | Plugin system and dependency groups stabilized[^5]. |
| 1.3–1.8 | 2022–2024 | Lock format changes, `poetry export` moved to a plugin, resolver and metadata fixes. |
| 2.0.0 | 2025-01 | First-class PEP 621 `[project]` table support; poetry-core 2.0[^4]. |

## References

[^1]: Poetry 1.0.0 release. https://github.com/python-poetry/poetry/releases/tag/1.0.0
[^2]: Poetry resolver overview and PubGrub/mixology background. https://python-poetry.org/docs/dependency-specification/
[^3]: PEP 621 — Storing project metadata in pyproject.toml. https://peps.python.org/pep-0621/
[^4]: Poetry 2.0 release notes — `[project]` table support. https://python-poetry.org/blog/announcing-poetry-2.0.0/
[^5]: poetry-plugin-export — export to requirements.txt. https://github.com/python-poetry/poetry-plugin-export

## Tags

python, packaging, dependency-management, package-manager, pyproject-toml, lockfile, build-backend, virtualenv, pep-621, cli, developer-tools
