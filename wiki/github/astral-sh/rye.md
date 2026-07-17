# astral-sh/rye

> An all-in-one Python project manager written in Rust — now archived and superseded by uv from the same team.

[GitHub repo](https://github.com/astral-sh/rye) ·
[Official website](https://rye.astral.sh) ·
[License: MIT](https://github.com/astral-sh/rye/blob/main/LICENSE)

## Overview

Rye is a project and package management tool for Python that tries to be the single entry point for everything: installing Python interpreters, creating and managing `pyproject.toml` projects, resolving and locking dependencies, managing virtualenvs, building wheels, and publishing to PyPI. It was started in April 2023 by Armin Ronacher (mitsuhiko, creator of Flask and Jinja) as a personal experiment in what a Cargo-like, batteries-included Python workflow could look like[^1]. The repository has since moved under the `astral-sh` organization — the same company behind ruff and uv — after Ronacher handed stewardship to Astral in early 2024[^2].

The defining fact about Rye today is that **it is no longer developed**. The maintainers explicitly direct users to uv, the successor project from the same team, and state that no further updates are planned — including security updates[^3]. The GitHub repository is archived. Rye's own documentation ships a uv migration guide. Any evaluation of Rye in 2026 is therefore an evaluation of a frozen tool, useful mainly as historical context and as an on-ramp to uv rather than as something to adopt fresh.

Architecturally Rye is best understood as an orchestrator, not an implementation. Rather than writing its own resolver, downloader, or formatter, it picks existing tools and wires them together behind one CLI. That design is what let a single author cover the whole Python workflow quickly, and it is also why the project could be gracefully retired: most of the components it delegated to (uv, ruff) are maintained independently and continue on their own.

## Getting Started

Installation is via a shell script on Unix or a downloadable installer on Windows[^4]:

```bash
# Linux and macOS
curl -sSf https://rye.astral.sh/get | bash
```

```bash
# create a new project (writes pyproject.toml, .python-version, src/ layout)
rye init my-app
cd my-app

# pin an interpreter Rye will fetch and manage for you
rye pin 3.12

# add dependencies and sync the managed virtualenv + lockfile
rye add requests
rye sync

# run code inside the managed environment
rye run python -c "import requests; print(requests.__version__)"
```

`rye sync` writes two lockfiles (`requirements.lock` and `requirements-dev.lock`) and materializes a `.venv` in the project. Because Rye manages the interpreter itself, no system Python is required.

## Architecture / How It Works

Rye is a Rust binary that shells out to and bundles a set of Python-ecosystem tools, presenting them under one command surface[^5]:

- **Interpreter provisioning** — Rye downloads standalone CPython builds from the `python-build-standalone` project (Indygreg builds) and PyPy distributions, rather than relying on a system Python or `pyenv`. A `.python-version` file records the pin per project.
- **Shims** — Rye installs `python`, `python3`, and tool shims onto `PATH`. When you invoke `python` inside a Rye-managed directory, the shim resolves to the project's pinned interpreter and virtualenv. This is the mechanism that makes "the right Python" automatic but is also the part most likely to conflict with an existing `pyenv`/`asdf`/system setup.
- **Dependency resolution and install** — originally implemented with `unearth` + `pip-tools`, later switched to **uv** as the default backend for locking and installation once uv existed, with the older path kept as a fallback.
- **Virtualenvs** — created via the standard `virtualenv` library.
- **Linting and formatting** — `rye lint` and `rye fmt` are thin wrappers over a bundled **ruff**.
- **Building and publishing** — wheel builds are delegated to the PyPA `build` package; `rye publish` uses `twine`.

The project intentionally stays close to standards: it is `pyproject.toml`-centric, supports workspaces (multiple libraries in one repo sharing a lockfile), and supports global tool installation similar to `pipx`. The tradeoff is that Rye's behavior is the sum of its bundled tools' behaviors — its resolver semantics, for example, became uv's semantics once uv was adopted.

## Production Notes

- **It receives no updates, including security fixes.** This is the operative caveat. Pinning a build for reproducibility is fine; treating Rye as a maintained tool is not. New Python releases, PyPI metadata changes, or CVEs in bundled components will not be addressed upstream[^3].
- **Shims are the most common footgun.** Rye prepends its shim directory to `PATH`. On machines that already use `pyenv`, `asdf`, Homebrew Python, or conda, ordering conflicts produce "wrong Python" surprises. Installing Rye on a clean shell profile and understanding the `PATH` ordering is worth doing before debugging environment issues.
- **The lock format is Rye's own.** `requirements.lock` is a flat, `pip`-style file, not a cross-platform universal lock. Migrating to uv means regenerating locks with uv's format (`uv.lock`), which is not a drop-in rename.
- **Migration path is real but not free.** The uv migration guide covers `pyproject.toml` reuse, but Rye-specific config under `[tool.rye]` (sources, managed scripts, workspace members) needs manual translation to uv equivalents. Budget time for CI changes since the invoked commands differ.
- **`rye sync` and CI.** Because Rye fetches interpreters over the network, cold CI runs download a full CPython build. Caching the Rye home directory (`~/.rye`) is the standard mitigation, but since the tool is frozen, most teams should instead cut over to uv in CI rather than harden a Rye pipeline.

## When to Use / When Not

**Use when:**
- You already have an existing Rye-managed project and need to keep it running short-term while planning a uv migration.
- You are studying how an all-in-one Python manager was assembled, as background before adopting uv.

**Avoid when:**
- You are starting anything new — use uv directly.
- You need ongoing maintenance, security updates, or new Python version support.
- You want a tool with an independent, still-maintained governance story (Poetry, PDM).

## Alternatives

- astral-sh/uv — the official successor from the same team; use this instead of Rye in every new case, and as the migration target for existing Rye projects.
- python-poetry/poetry — mature, independently maintained project + dependency manager with its own resolver and lock format; use when you want a long-established tool not tied to Astral.
- pdm-project/pdm — standards-forward project manager (PEP 621 metadata, pluggable backends); use when you want a lighter, spec-driven alternative to Poetry.
- pypa/pipenv — older Pipfile-based application dependency workflow; use when maintaining a legacy Pipenv project, otherwise superseded.
- pyenv/pyenv — if the only thing you actually need is interpreter version management (not projects or locking), pyenv is narrower and still maintained.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2023-04 | Initial release by Armin Ronacher as an experimental Python project manager[^1]. |
| — | 2023-05 | Public introduction; "Rye: An Experimental Package Management Solution for Python"[^1]. |
| — | 2024-02 | Stewardship moved to Astral; uv released and progressively adopted as Rye's locking/install backend[^2]. |
| — | 2024-08 | "Harvest Season" — Rye positioned as effectively superseded by uv; project enters wind-down[^6]. |
| 0.4x | 2025 | Final maintenance-era releases; feature work stops, users pointed to uv[^3]. |
| archived | 2026-02 | Repository archived; no further updates planned, including security updates[^3]. |

## References

[^1]: Armin Ronacher, "Rye: An Experimental Package Management Solution for Python" — 2023. https://lucumr.pocoo.org/2023/5/6/potw-rye/
[^2]: Astral, "Rye Grows With Astral" — 2024. https://astral.sh/blog/uv
[^3]: Rye README / documentation notice: "Rye is no longer developed... no further updates are planned, including security updates." https://github.com/astral-sh/rye
[^4]: Rye installation guide. https://rye.astral.sh/guide/installation/
[^5]: Rye documentation, "Philosophy" and tool composition (python-build-standalone, uv, ruff, virtualenv, build, twine). https://rye.astral.sh/guide/
[^6]: Armin Ronacher, "Harvest Season" — 2024-08-21. https://lucumr.pocoo.org/2024/8/21/harvest-season/

## Tags

python, package-manager, packaging, dependency-management, virtualenv, rust, cli, astral, deprecated, developer-tools
