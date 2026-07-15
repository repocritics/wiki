# pypa/hatch

> Python project manager whose build backend (Hatchling) is now a de facto PyPA standard, even for teams that never touch the CLI.

[GitHub repo](https://github.com/pypa/hatch) ·
[Official website](https://hatch.pypa.io/latest/) ·
[License: MIT](https://spdx.org/licenses/MIT.html)

## Overview

Hatch is two things shipped from one repository. **Hatchling** is a PEP 517 build backend — the piece that turns a source tree into a wheel and sdist. **Hatch** is the workflow CLI on top of it: environment management, matrix test runners, version bumping, a script runner, publishing, and Python interpreter installation. The project was started by Ofek Lev in 2017[^1], rewritten from the ground up for its 1.0 release in 2022[^2], and is maintained under the Python Packaging Authority (PyPA) umbrella.

The defining fact about Hatch is that its two halves have had very different fates. Hatchling won the build-backend contest: it is the backend the official Python packaging tutorial reaches for, has a minimal dependency footprint, cleanly implements PEP 621 metadata in `[project]`, and is used by a large share of modern pure-Python and mixed projects — often by teams whose day-to-day tool is pip, uv, or Poetry and who never run `hatch` at all. The `hatch` CLI, by contrast, is one of several competing workflow managers (Poetry, PDM, and now uv) and has not achieved the same dominance.

The tension that follows from this: Hatch is standards-first and unopinionated about dependency resolution. It has historically had **no lockfile** of its own, delegating installation to pip (or, more recently, uv). That is exactly why some teams love it (no bespoke resolver, no vendor lock-in, plays nicely with the PEP ecosystem) and why others skip it (application developers who want reproducible, hash-pinned installs reach for a lockfile-first tool instead).

## Getting Started

```bash
pipx install hatch      # isolated global install (recommended)
# or: pip install hatch
hatch new "My App"
cd my-app
hatch run test:cov      # runs the "cov" script in the "test" environment matrix
hatch build             # produces sdist + wheel in dist/
```

Using Hatchling as just a build backend — no `hatch` CLI required — is the more common pattern. A minimal `pyproject.toml`:

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-app"
dynamic = ["version"]
requires-python = ">=3.9"
dependencies = ["httpx"]

[tool.hatch.version]
path = "src/my_app/__about__.py"   # single source of truth for the version
```

Any front end that speaks PEP 517 (`pip install .`, `python -m build`, uv, Poetry) will drive that backend without Hatch being installed.

## Architecture / How It Works

Hatchling and Hatch are separate distributions built from the same monorepo, and understanding them separately is the key to using the project well.

**Hatchling (the backend).** It implements the PEP 517 hooks (`build_wheel`, `build_sdist`, editable installs) and reads PEP 621 metadata from `[project]`. Its power is a plugin system wired through entry points: *builders* (wheel, sdist, and custom targets), *build hooks* (run code at build time — e.g. compiling assets), *metadata hooks* (compute metadata dynamically), and *version source* plugins. Hatchling itself keeps a deliberately small dependency set, which is a large part of why it is trusted as a base for other projects.

**Hatch (the CLI).** Its central abstraction is the **environment**. Environments are declared under `[tool.hatch.envs.*]` in `pyproject.toml`, each an isolated, cached virtualenv with its own dependencies and scripts. A **matrix** environment fans out across Python versions or dependency variants (e.g. test on 3.9–3.13 in one command). By default environments are provisioned with virtualenv + pip; Hatch can also use **uv** as the installer/resolver for large speed gains[^3].

**Version management.** `hatch version` reads and writes the project version from a configured source — a regex against a file, or a plugin. The common `hatch-vcs` plugin derives the version from Git tags (via setuptools-scm underneath), so releases are driven by tagging rather than editing a file[^4].

**Python distribution management.** `hatch python install` downloads standalone CPython builds (the python-build-standalone distributions) so environments can target interpreter versions that are not installed system-wide[^5].

The couplings are loose by design. The CLI depends on the backend, but not the reverse; you can adopt Hatchling and later choose a completely different workflow tool, or vice versa, without a rewrite.

## Production Notes

**No native lockfile.** This is the single most important operator caveat. Hatch environments install from your declared dependencies each time; there is no Hatch-managed lockfile pinning exact versions and hashes. For libraries this is usually fine (you want a range, not a pin). For deployed applications that need byte-for-byte reproducibility, you must layer something on top — `pip-tools`/`pip compile`, `uv pip compile`, or a separate lockfile tool — or pick a lockfile-first manager instead. Do not assume `hatch env` gives you reproducible installs.

**Environment creation cost.** Matrix environments across several Python versions create several virtualenvs, and with the default pip installer this can be slow in CI. Switching the installer to uv (`installer = "uv"`) cuts environment build time substantially and is the standard mitigation for slow pipelines[^3].

**Global vs. project state.** Environments are cached outside the project tree (under a Hatch data directory), not in a `.venv` next to your code the way some tools default. This trips up developers expecting a local `.venv`; use `hatch env find` to locate an environment, or configure the env `type`/location explicitly. IDE integration is less automatic than with tools that create a project-local venv.

**Adopting Hatchling on non-trivial builds.** Pure-Python packages are frictionless. Anything with compiled extensions, data files, or generated code needs explicit `[tool.hatch.build]` configuration (`force-include`, `artifacts`, `packages`, build hooks). The defaults exclude files not tracked by your VCS, which surprises people whose generated artifacts are gitignored — they silently vanish from the wheel.

**Upgrade posture.** Hatch and Hatchling version independently. Because Hatchling is a build-time dependency pinned in `[build-system].requires`, a Hatchling change can affect anyone building your package even if they never installed Hatch — pin a floor and test builds when bumping.

## When to Use / When Not

**Use when:**
- You want a standards-compliant, low-dependency build backend — Hatchling is a safe default for most Python packages.
- You publish libraries and want clean PEP 621 metadata, VCS-based versioning, and simple `build`/`publish`.
- You need matrix testing across Python versions without hand-rolling tox.
- You want to stay tool-agnostic and avoid a bespoke resolver/lockfile format.

**Avoid when:**
- You need a lockfile with hash-pinned, fully reproducible application installs out of the box — reach for uv, Poetry, or PDM.
- You want a project-local `.venv` and zero-config IDE pickup by default.
- Your team wants one opinionated tool that resolves, locks, and runs everything with no assembly — uv now covers that ground.

## Alternatives

- astral-sh/uv — use instead when you want an extremely fast all-in-one resolver, lockfile, and runner; increasingly the default choice for applications.
- python-poetry/poetry — use instead when you want mature, lockfile-first dependency management for applications with a large ecosystem of guides.
- pdm-project/pdm — use instead when you want PEP 621 standards plus a lockfile and PEP 582/centralized installs.
- pypa/flit — use instead when you want the simplest possible build backend for a single pure-Python module or package.
- pypa/setuptools — use instead when you have legacy C-extension builds or `setup.py` logic that Hatchling's declarative model does not cover.

## History

| Version | Date | Notes |
|---------|------|-------|
| pre-1.0 | 2017 | Original Hatch created by Ofek Lev[^1]. |
| 1.0 | 2022 | Complete rewrite; Hatchling split out as a standalone build backend; PEP 621 metadata[^2]. |
| 1.x | 2022–2023 | Adopted under PyPA; script runner, project templates, static-analysis defaults. |
| 1.x | 2023–2024 | Python distribution management (`hatch python`) via standalone CPython builds[^5]. |
| 1.x | 2024 | uv supported as environment installer/resolver[^3]. |

## References

[^1]: Hatch history — original project by Ofek Lev, first released 2017. https://hatch.pypa.io/latest/history/hatch/
[^2]: Hatch 1.0 rewrite and Hatchling backend. https://hatch.pypa.io/latest/history/hatch/
[^3]: Hatch environment configuration — installer selection (pip / uv). https://hatch.pypa.io/latest/config/environment/overview/
[^4]: `hatch-vcs` — VCS-based version source plugin (setuptools-scm). https://github.com/ofek/hatch-vcs
[^5]: Hatch Python distribution management tutorial. https://hatch.pypa.io/latest/tutorials/python/manage/

## Tags

python, packaging, build-backend, project-management, cli, pep-517, pep-621, virtualenv, versioning, pypa
