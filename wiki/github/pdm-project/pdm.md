# pdm-project/pdm

> A standards-first Python package and dependency manager — the tool that bet earliest on PEP 621 and PEP 582.

[GitHub repo](https://github.com/pdm-project/pdm) ·
[Official website](https://pdm-project.org) ·
[License: MIT](https://github.com/pdm-project/pdm/blob/main/LICENSE)

## Overview

PDM is a Python package and dependency manager created by Frost Ming, first released in 2019[^1]. Its stated design goal is to follow official packaging standards (PEPs) rather than invent parallel formats. It stores project metadata in a standard `[project]` table in `pyproject.toml` (PEP 621), builds distributions through a pluggable PEP 517 backend, and locks dependencies into a `pdm.lock` file. As of 2026 it sits in the second tier of Python project managers — well-maintained and standards-clean, but with a fraction of the mindshare of Poetry before it and uv after it. The roughly 8.7k stars and steady commit cadence (last push July 2026) describe a healthy, still-active project rather than a runaway one.

PDM's original defining bet was **PEP 582** — a `__pypackages__` local install directory that let you run Python against project dependencies without activating a virtualenv, analogous to Node's `node_modules`. That bet did not pay off: the Python Steering Council rejected PEP 582 in early 2023[^2], and PDM demoted it to an opt-in experimental mode with virtualenvs as the default. What survived, and aged well, was the standards discipline: PDM shipped PEP 621 `[project]` metadata years before Poetry adopted it (Poetry used its own `[tool.poetry]` table until Poetry 2.0 in 2025), so a `pyproject.toml` written for PDM is far more portable than one written for older Poetry.

The defining tension today is positional. PDM is more standards-native than Poetry and more full-featured (lockfile, scripts, plugins) than a bare `pip` + `venv`, but the arrival of astral-sh/uv — dramatically faster and now the momentum leader — has narrowed the reason to pick PDM specifically. PDM's own answer is to integrate uv as an optional backend rather than compete with it head-on.

## Getting Started

```bash
# standalone installer (recommended — isolates PDM from your project env)
curl -sSL https://pdm-project.org/install.sh | bash
# or: pipx install pdm  /  brew install pdm
```

```bash
pdm new my-project          # scaffold pyproject.toml interactively
cd my-project
pdm add requests flask      # resolve, lock (pdm.lock), and install
pdm run python -m my_project
```

A minimal PEP 621 `pyproject.toml` that PDM produces and consumes:

```toml
[project]
name = "my-project"
version = "0.1.0"
requires-python = ">=3.10"
dependencies = ["requests>=2.31", "flask>=3.0"]

[build-system]
requires = ["pdm-backend"]
build-backend = "pdm.backend"
```

## Architecture / How It Works

PDM is a workflow layer over the standard Python packaging stack rather than a reimplementation of it:

- **Metadata** lives in the PEP 621 `[project]` table. PDM-specific settings (scripts, sources, resolution overrides) live under `[tool.pdm]`, keeping the portable metadata clean.
- **Resolution** uses `resolvelib` — the same backtracking resolver library pip uses — driven by PDM's own candidate/repository layer. The lockfile `pdm.lock` records pinned versions plus hashes, and can capture cross-platform locks so one lockfile serves multiple OSes and Python versions.
- **Build backend** is pluggable. The default is `pdm-backend` (formerly `pdm-pep517`), but because PDM only speaks the PEP 517 interface, you can swap in `hatchling`, `setuptools`, `flit-core`, or `maturin` without leaving PDM. This build-backend agnosticism is a genuine differentiator from Poetry, which is coupled to its own backend.
- **Environments.** PDM manages virtualenvs in either a project-local `.venv` or a centralized location keyed by interpreter, and can also install interpreters directly via astral-sh's python-build-standalone. PEP 582 `__pypackages__` mode still exists behind `pdm config python.use_venv false`.
- **Install cache.** With the centralized cache enabled, packages are unpacked once and linked (hard links / symlinks) into each project, in the spirit of pnpm — saving disk and install time across many projects.
- **Plugins** are ordinary Python distributions that register against PDM's entry points, so a plugin ships and installs like any other package.

The newest structural change is the **uv backend**: `pdm config use_uv true` delegates resolution and installation to uv while PDM keeps owning the project model, lockfile schema, and CLI. This makes PDM a front-end over uv's Rust engine — with the caveat that uv and PEP 582 are mutually exclusive.

## Production Notes

**Speed relative to uv.** PDM's pure-Python resolver and installer are competitive with Poetry and pip but not with uv. Large or conflict-heavy dependency graphs are where resolution time is felt. Teams that need speed increasingly enable the uv backend; at that point uv's own resolver semantics apply, and edge-case resolutions may differ from PDM's native path.

**PEP 582 is effectively a dead end.** It works, but it is a rejected PEP, off by default, and incompatible with the uv backend. New projects should treat virtualenv mode as the only supported path; `__pypackages__` should be considered legacy.

**Lockfile portability, with limits.** `pdm.lock` is a PDM-specific format, not a standard — a project locked with PDM cannot hand its lockfile to Poetry or uv. The PEP 751 `pylock.toml` standard aims to fix this ecosystem-wide; until it is universally adopted, the lockfile is a lock-in point even though the metadata is not. PDM can export to `requirements.txt` (`pdm export`) as an escape hatch for CI systems and Docker layers that only speak pip.

**Interpreter selection surprises.** PDM resolves which Python to use from `requires-python`, the active venv, and configured interpreters. On machines with many Pythons (pyenv, system, Homebrew, python-build-standalone) it is worth pinning explicitly with `pdm use` to avoid silently building or locking against an unintended interpreter.

**Plugin fragility on upgrade.** Because plugins hook internal entry points, a PDM minor upgrade can break a third-party plugin whose interface assumptions shifted. Pin plugin versions in CI and treat PDM upgrades as a coordinated step, not an automatic one.

**Standalone installer vs pip install.** Installing PDM into the same environment as your project (`pip install pdm`) risks dependency conflicts between PDM's own deps and your app's. The standalone installer / pipx path keeps PDM isolated and is the recommended production posture.

## When to Use / When Not

**Use when:**
- You want a lockfile and managed workflow but insist on standards-compliant PEP 621 metadata that stays portable.
- You need freedom to choose your build backend (native extensions via maturin, or hatchling) inside one tool.
- You maintain many projects and want pnpm-style centralized caching to save disk and install time.
- You want a migration path onto uv's speed without rewriting your project model today.

**Avoid when:**
- Raw speed and the current ecosystem momentum are decisive — uv is faster and now the default many teams reach for.
- Your team is already standardized on Poetry or Hatch and has no concrete pain to solve.
- You want zero third-party tooling: `pip` + `venv` + a hand-written `pyproject.toml` covers simple libraries.
- You depend heavily on PEP 582 semantics — that path is deprecated and unsupported alongside newer features.

## Alternatives

- astral-sh/uv — use instead when installation and resolution speed dominate; Rust-based, now the momentum leader, and usable as PDM's own backend.
- python-poetry/poetry — use instead when you want the largest community and tutorial base; Poetry 2.0 finally added PEP 621 support, narrowing PDM's standards edge.
- pypa/hatch — use instead when you want a PyPA-affiliated tool with strong multi-environment management and are comfortable with lighter lockfile support.
- pypa/pipenv — use instead for deploying non-installable applications (Django sites, services) where you never build a wheel.
- conda/conda — use instead when you need non-Python binary dependencies (CUDA, MKL, scientific stacks) that PyPI wheels do not cover.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2019-12 | Initial release; PEP 582 `__pypackages__` as the headline feature[^1]. |
| 1.0 | 2021 | First stable line; PEP 621 metadata, `pdm.lock`, plugin system. |
| 2.0 | 2022 | Major workflow overhaul; virtualenv-centric defaults. |
| 2.x | 2023 | PEP 582 demoted after Steering Council rejection[^2]; standalone installer, python-build-standalone integration. |
| 2.x | 2024–2026 | uv backend integration (`use_uv`), continued PEP tracking; last push 2026-07[^3]. |

## References

[^1]: PDM repository and documentation, "What is PDM?". https://pdm-project.org
[^2]: Python Steering Council, "PEP 582 rejection" (2023). https://discuss.python.org/t/pep-582-python-local-packages-directory/963/430
[^3]: pdm-project/pdm on GitHub — repository metadata (stars, forks, last push) as of 2026-07. https://github.com/pdm-project/pdm

## Tags

python, package-manager, dependency-management, packaging, pep621, pep517, pep582, lockfile, pyproject-toml, cli, virtualenv
