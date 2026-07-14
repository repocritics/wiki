# astral-sh/uv

> An extremely fast Python package and project manager, written in Rust — one binary meant to replace pip, pip-tools, pipx, poetry, pyenv, twine, and virtualenv.

[GitHub repo](https://github.com/astral-sh/uv) ·
[Official website](https://docs.astral.sh/uv) ·
[License: Apache-2.0 OR MIT](https://github.com/astral-sh/uv/blob/main/LICENSE-APACHE)

## Overview

uv is a Python packaging tool from Astral, the company behind the Ruff linter, first released in February 2024[^1]. It began as a drop-in reimplementation of pip and pip-tools in Rust, and by mid-2024 grew into a full project and environment manager: dependency resolution, lockfiles, virtual environments, Python interpreter installation, and tool execution, all from a single static binary that needs neither Python nor Rust to bootstrap[^2].

Its defining claim is speed — typically 10-100x faster than pip for installs with a warm cache[^3] — achieved through a Rust resolver (built on PubGrub), aggressive parallelism, and a global content-addressed cache that hardlinks packages across environments instead of recopying them. The practical draw, though, is consolidation: Python's tooling had fragmented across pip, virtualenv, pyenv, pipx, poetry, pdm, and hatch, and uv folds most of those workflows into one command surface.

The central tension is governance versus convenience. uv is fast, well-engineered, and pleasant, but it is a pre-1.0 (0.x) tool from a single venture-backed company, and it reintroduces a vendor-centric dependency into an ecosystem that spent a decade standardizing around community PEPs and the PyPA. It also ships its own Python builds and its own lockfile format rather than the interpreter and standards a user might already assume. For many teams the ergonomics win outright; for others the concentration of the toolchain under one commercial vendor is the thing to weigh.

## Getting Started

```bash
# Standalone installer (no Python required)
curl -LsSf https://astral.sh/uv/install.sh | sh
# Windows: powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
# or: pip install uv  /  pipx install uv
```

```bash
uv init example && cd example      # scaffold a project (pyproject.toml)
uv add requests                    # resolve, create .venv, write uv.lock
uv run python -c "import requests" # run inside the managed environment
uv lock --upgrade                  # re-resolve to newest allowed versions
uv sync --frozen                   # install exactly what the lockfile pins
```

```bash
# pip-compatible interface for incremental adoption
uv venv                                    # create a virtualenv
uv pip install -r requirements.txt         # familiar pip semantics, faster
uv pip compile requirements.in --universal -o requirements.txt
```

## Architecture / How It Works

uv is a single Rust binary exposing several loosely related command groups:

- **Project interface** (`uv init/add/remove/lock/sync/run`) — manages a `pyproject.toml` and a `uv.lock` universal lockfile. The lockfile resolves for *all* platforms and Python versions at once, so one `uv.lock` produces reproducible installs on macOS, Linux, and Windows without re-resolving per machine.
- **pip interface** (`uv pip install/compile/sync`) — deliberately mimics pip and pip-tools semantics for teams that want the speedup without adopting the project model. It is a separate, lower-level surface: it does not read or write `uv.lock`.
- **Tools** (`uv tool install`, and `uvx` as an alias for `uv tool run`) — the pipx equivalent, running or installing CLI packages in isolated environments.
- **Python management** (`uv python install/pin`) — downloads standalone CPython (and PyPy) builds on demand rather than using or compiling a system interpreter.

The resolver is built on **PubGrub**[^4], a version-solving algorithm that produces clear, minimal conflict explanations instead of pip's older backtracking. Resolution, downloading, and unpacking run in parallel. Packages are stored once in a **global cache** and materialized into each `.venv` via hardlinks or copy-on-write clones, so N projects sharing a dependency cost roughly one copy of it on disk.

Two design choices have outsized downstream effects. First, uv installs its **own** Python distributions — the `python-build-standalone` project, which Astral now stewards — so the interpreter is a relocatable prebuilt binary, not the distro's Python. Second, the `uv.lock` format is uv-specific; it is not the PyPA's emerging `pylock.toml` (PEP 751) standard, though uv can export to `requirements.txt` and to `pylock.toml`.

## Production Notes

**It is still 0.x.** uv follows a documented versioning policy but has not committed to 1.0 semantics. Breaking changes have been infrequent and well-flagged, but you should pin the uv version itself in CI (the installer supports pinning) rather than always pulling latest.

**The global cache assumes a shared filesystem.** Hardlinking only works when the cache and the target `.venv` sit on the *same* filesystem. In Docker, on network mounts, or when `/tmp` and the project live on different volumes, uv silently falls back to copying — you lose both the disk-dedup and much of the speed. Set `UV_LINK_MODE=copy` explicitly to make that behavior deterministic, and mount the cache (`UV_CACHE_DIR`) as a build cache in containers.

**Managed Python is not system Python.** `python-build-standalone` interpreters are built with specific flags and are statically relocatable; most code doesn't notice, but native extensions that hardcode library paths, tools that shell out to a distro `python3`, or environments expecting a system OpenSSL can behave differently. If you need the platform interpreter, pass `--python-preference only-system`.

**Docker.** The common pattern is a multi-stage build using Astral's `ghcr.io/astral-sh/uv` image (or copying the binary in), `UV_COMPILE_BYTECODE=1` for faster cold starts, `UV_LINK_MODE=copy`, and `uv sync --frozen --no-dev` against a cache mount. `--frozen` (fail if the lock is stale) and `--locked` are the reproducibility guardrails for CI.

**Lockfile portability.** `uv.lock` is only consumable by uv. If your consumers (other tooling, security scanners, non-uv CI) need a standard artifact, generate one with `uv export --format requirements-txt` or `uv pip compile`. Do not assume other Python tools understand the native lock.

**Ecosystem coupling.** Betting a whole org's packaging on uv means betting on Astral's continued open-source stewardship of a VC-backed company. This is a governance question, not a code-quality one — uv is Apache/MIT dual-licensed and the resolver internals are open — but it is the objection most worth naming before standardizing on it.

## When to Use / When Not

**Use when:**
- You want fast, reproducible installs and a single tool spanning venvs, resolution, lockfiles, Python versions, and tool execution.
- You are starting a new project and want a modern, opinionated workflow (`uv init`/`add`/`sync`).
- You want to accelerate existing pip/pip-tools workflows with minimal change via `uv pip`.
- CI install time or Docker image build time is a real cost you want to cut.

**Avoid when:**
- You require a finalized 1.0 tool with a long back-compat guarantee, or your policy forbids non-standard lockfile formats.
- Your org is wary of consolidating the toolchain under one commercial vendor.
- You depend on distro-managed system Python and native builds that assume it.
- You are deep in an existing poetry/pdm setup that works and gains little from switching.

## Alternatives

- python-poetry/poetry — mature project/dependency manager; use instead when you want a settled, widely-integrated tool and don't need uv's speed.
- pypa/pip — the reference installer; use instead when you want zero extra dependencies and strict PyPA-standard behavior.
- pdm-project/pdm — PEP-standards-first project manager; use instead when standard `pylock.toml`/PEP 621 fidelity matters more than raw speed.
- astral-sh/rye — Astral's earlier project manager, now effectively in maintenance and steering users toward uv; new projects should generally pick uv.
- prefix-dev/pixi — Rust-based manager built on the conda ecosystem; use instead when you need conda packages and non-Python (C/CUDA) dependencies.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.0 | 2024-02-15 | Initial release: Rust reimplementation of pip / pip-tools plus `uv venv`[^1]. |
| 0.2 | 2024-05 | Resolver and pip-interface hardening; overrides, universal resolution. |
| 0.3 | 2024-08 | "Unified" launch: project management (`init`/`add`/`sync`/`lock`), tools (`uvx`), managed Python, workspaces[^2]. |
| 0.4 | 2024-09 | Broader Python version management and install ergonomics. |
| 0.5 | 2024-11 | Stabilization of project workflows; toward a stable lockfile. |
| 0.6 | 2025-02 | `pylock.toml` (PEP 751) export/interop; continued lock format work[^5]. |

## References

[^1]: Astral, "uv: Python packaging in Rust" — 2024-02-15. https://astral.sh/blog/uv
[^2]: Astral, "uv: Unified Python packaging" — 2024-08-20. https://astral.sh/blog/uv-unified-python-packaging
[^3]: uv benchmarks. https://github.com/astral-sh/uv/blob/main/BENCHMARKS.md
[^4]: PubGrub version-solving algorithm (Rust implementation). https://github.com/pubgrub-rs/pubgrub
[^5]: PEP 751 — A file format to record Python dependencies for installation reproducibility. https://peps.python.org/pep-0751/
[^6]: uv documentation. https://docs.astral.sh/uv

## Tags

rust, python, packaging, dependency-management, package-manager, virtualenv, resolver, cli, developer-tools, astral
