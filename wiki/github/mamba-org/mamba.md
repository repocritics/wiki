# mamba-org/mamba

> A C++ reimplementation of the conda package manager — fast parallel downloads and libsolv-based solving, plus the standalone micromamba binary.

[GitHub repo](https://github.com/mamba-org/mamba) ·
[Official website](https://mamba.readthedocs.io) ·
[License: BSD-3-Clause](https://github.com/mamba-org/mamba/blob/main/LICENSE)

## Overview

Mamba is a drop-in-compatible reimplementation of the conda package manager, written in C++ and originally built by QuantStack as part of the conda-forge ecosystem[^1]. Its original pitch was speed: conda's classic dependency solver was slow enough that installing a scientific stack could take minutes, and mamba replaced it with libsolv (the SAT-based solver behind Red Hat, Fedora and openSUSE package tooling) plus multi-threaded downloads of repository metadata and packages. For years mamba was the default recommendation whenever conda "felt slow."

The project ships two front-ends. `mamba` is the full CLI, dynamically linked and meant to sit alongside a conda/miniforge installation, sharing channels and the `base` environment. `micromamba` is a single statically linked executable with no Python dependency — it can bootstrap an environment from nothing, which made it the standard tool for CI pipelines and Docker images where installing full conda first is wasteful.

The defining tension in 2026 is that mamba partly won its own argument. The solver engine, `libmamba`, was upstreamed into conda and became conda's **default** solver in conda 23.10 (October 2023)[^2]. Modern conda now solves with the same libsolv-based engine, so the headline speed gap has narrowed to parallel downloading and the small-footprint micromamba binary rather than solving time. Mamba remains widely used, but "use mamba because conda's solver is slow" is a weaker reason than it was.

## Getting Started

```bash
# Self-contained micromamba binary (no conda required)
"${SHELL}" <(curl -L micro.mamba.pm/install.sh)

# Or install mamba into an existing conda base env
conda install -n base -c conda-forge mamba
```

```bash
# Create and use an environment (micromamba syntax mirrors mamba/conda)
micromamba create -n sci -c conda-forge python=3.12 numpy pandas
micromamba activate sci
python -c "import numpy; print(numpy.__version__)"
```

## Architecture / How It Works

Mamba's speed comes from three parts working together:

- **libsolv** — the dependency resolution core. Repository index files (`repodata.json`) are loaded into libsolv's in-memory pool and the install request is expressed as a SAT problem. This is the same battle-tested solver used by `zypper`/`dnf`, and it is what conda eventually adopted.
- **Parallel I/O** — repository metadata and package tarballs are fetched with a multi-threaded curl-based downloader, rather than conda's historically serial fetches.
- **`libmamba` / `libmambapy`** — the C++ engine is exposed as a shared library with Python bindings, which is exactly what allowed conda to embed it as `conda-libmamba-solver`.

Historically mamba 1.x was a thin front-end: it reused conda's own Python code for command parsing, transaction verification, and package (de)installation, and only swapped in the C++ solver and downloader. This kept behavior close to conda but coupled mamba to conda's internals. Mamba 2.0 restructured this — `mamba` and `micromamba` were unified onto a single C++ codebase, reducing the dependence on conda's Python layer[^3]. That rewrite is the source of most 1.x → 2.x migration friction.

`micromamba` differs from `mamba` in packaging, not engine: it links libsolv, libcurl and the mamba code statically into one executable, so it has no runtime dependency on an existing conda/Python installation. It uses its own shell hook (`micromamba shell init`) and its own `micromamba activate`, rather than `conda activate`.

## Production Notes

- **The solver is no longer a differentiator vs. modern conda.** Since conda 23.10 both tools solve with libsolv. If your reason for adopting mamba is solve speed on a recent conda, benchmark first — the win is now mostly parallel downloads and micromamba's tiny footprint, not solving[^2].
- **micromamba is not `conda`.** It does not read a conda install's shell activation; you must run `micromamba shell init` and use `micromamba activate`. Scripts that hardcode `conda activate` will not work unchanged. This is the most common CI footgun.
- **1.x is effectively frozen.** Only mamba/micromamba 2.0 and later are actively developed; the 1.x branch is maintained only for security fixes (CVEs)[^1]. New projects should target 2.x.
- **1.x → 2.x has behavior changes.** The 2.0 rewrite changed CLI details, output formatting, and some defaults. Pin versions in reproducible pipelines and test upgrades rather than floating `latest`.
- **Channel priority matters.** Mixing `defaults` (Anaconda) and `conda-forge` without strict channel priority produces subtly different solves and can trip Anaconda's commercial ToS on the `defaults` channel — most users set `conda-forge` as the sole channel with strict priority.
- **pip interop is partial.** Like conda, mamba does not model pip-installed packages in its solver. `pip install` inside a mamba env works but is invisible to subsequent mamba solves and can be clobbered on the next environment change.
- **`env export` differs from conda.** Mamba normalizes `MatchSpec` strings to their simplest form, so `mamba env export` output is not byte-identical to `conda env export`[^1] — relevant if you diff or pin against conda-produced lockfiles.
- **libmamba C++ API is unstable.** Maintainers explicitly reserve the right to break the C++ API/ABI across minor/major releases, since they are unaware of external consumers[^1]. Do not build long-lived C++ integrations against it expecting stability; `libmambapy` gives stronger (minor-compatible) guarantees.

## When to Use / When Not

**Use when:**
- You need a small, dependency-free package manager for CI or Docker — micromamba installs in ~1 second and bootstraps environments without conda.
- You want conda-forge packages with fast parallel downloads.
- You are installing `conda-lock` lockfiles and want to avoid installing conda-lock itself (`micromamba create -f *-lock.yml`).
- You want the setup-micromamba GitHub Action as a faster setup-miniconda replacement.

**Avoid when:**
- You are already on a recent conda and adopted mamba only for solve speed — the gap has largely closed.
- You need pure-Python packaging from PyPI with maximum speed — uv/pip target that space better than the conda ecosystem.
- You want project/lockfile-first workflows with a modern UX — pixi (from the same authors) is designed for that.
- You depend on a stable C++ API — libmamba does not promise one.

## Alternatives

- conda/conda — the reference implementation mamba clones; now ships mamba's libsolv engine as its default solver, so speed parity is close.
- prefix-dev/pixi — Rust, project- and lockfile-first package manager from the original mamba authors (prefix.dev); use it instead when you want a modern `cargo`-like workflow over conda channels.
- conda-forge/miniforge — a conda/mamba distribution preconfigured for conda-forge; use it instead when you want a batteries-included installer rather than a bare binary.
- astral-sh/uv — extremely fast Python packaging/resolver for PyPI; use it instead when you only need Python wheels and not the broader conda binary ecosystem.
- pypa/pip — the PyPI installer; use it instead when your dependencies are all on PyPI and you do not need conda's non-Python binaries.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2019 | First public release; C++ + libsolv solver and parallel downloads, wrapping conda's Python for install/transaction code[^1]. |
| micromamba | 2020 | Statically linked standalone executable introduced for CI/containers. |
| 1.0 | 2022-11 | First stable major release; API/versioning stabilization[^4]. |
| (conda 23.10) | 2023-10 | conda adopts libmamba as its **default** solver, mainstreaming mamba's engine[^2]. |
| 2.0 | 2024 | Full C++ rewrite; `mamba` and `micromamba` unified on one codebase, reduced conda-Python coupling, breaking changes[^3]. |

## References

[^1]: mamba README and documentation. https://github.com/mamba-org/mamba — https://mamba.readthedocs.io/en/latest
[^2]: Anaconda, "conda 23.10.0 released: libmamba solver is now the default." https://www.anaconda.com/blog/a-faster-conda-for-a-growing-community
[^3]: mamba 2.0 documentation and migration notes. https://mamba.readthedocs.io/en/latest
[^4]: mamba releases and changelog. https://github.com/mamba-org/mamba/releases

## Tags

package-manager, conda, cpp, python, dependency-solver, libsolv, conda-forge, micromamba, cli, scientific-computing, ci-cd
