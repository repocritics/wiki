# conda/conda

> A language-agnostic, system-level binary package and environment manager — the runtime engine behind Anaconda, Miniconda, and conda-forge.

[GitHub repo](https://github.com/conda/conda) ·
[Official website](https://docs.conda.io/projects/conda/) ·
[License: BSD-3-Clause](https://github.com/conda/conda/blob/main/LICENSE)

## Overview

Conda is a cross-platform package and environment manager first released by Continuum Analytics (now Anaconda, Inc.) in 2012[^1]. Unlike pip, it is not Python-specific: it installs prebuilt binary packages for any language (Python, R, C/C++ libraries, CUDA toolkits, compilers) and treats environments as first-class, fully isolated installations rather than as `PYTHONPATH` overlays. This is why it became the default tool for scientific computing and machine learning, where the hard problem is not the Python code but the native shared libraries (MKL, OpenBLAS, CUDA, GDAL, HDF5) underneath it.

The command-line tool is written in Python and BSD-licensed, but "conda" in practice means three separable things that are easy to conflate: the `conda/conda` CLI (this repo), the *channels* it downloads from (`defaults`/`repo.anaconda.com`, run by Anaconda, Inc., versus the community-run `conda-forge`), and the *distributions* that bundle it (Anaconda Distribution, Miniconda, Miniforge). The defining tension of the ecosystem is that the tool is open source and free, but the most convenient package channel — Anaconda's `defaults` — carries commercial-use terms that changed in ways that surprised many organizations[^2].

Conda's second defining tension was performance: for most of its life the classic SAT-based dependency solver was slow enough that "conda is solving environment" became a running joke. That was substantially addressed in 2023 (see Architecture).

## Getting Started

Install a minimal distribution — Miniconda (Anaconda channels) or Miniforge (conda-forge, community-governed and free of Anaconda's channel terms):

```bash
# Miniforge — recommended if you want to avoid the defaults channel terms
curl -L -O https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh
bash Miniforge3-$(uname)-$(uname -m).sh
```

```bash
# Create an isolated environment and install cross-language deps
conda create --name ml python=3.12 pytorch numpy
conda activate ml
conda list          # inspect installed packages
conda deactivate    # return to the base environment
```

Environments live under `envs/` in the install prefix and are populated with hard links to a shared package cache when the filesystem allows, so creation is fast and space-efficient. Configuration lives in `~/.condarc` (channels, channel priority, solver).

## Architecture / How It Works

A conda package is a compressed archive (`.tar.bz2`, or the newer zstd-based `.conda` format introduced in conda 4.7[^3]) containing prebuilt binaries plus an `info/` directory of metadata: dependency constraints, the target platform (`linux-64`, `osx-arm64`, `win-64`, `noarch`), and a build string. Each channel publishes a `repodata.json` index listing every package and its constraints; conda downloads and caches these indexes, then must find a mutually consistent set of package versions.

That "find a consistent set" step is a boolean satisfiability (SAT) problem. The classic solver encoded constraints and handed them to `pycosat`. Correct, but on large channels (`conda-forge` has hundreds of thousands of records) it could take minutes and occasionally backtrack into pathological cases. In 2022 the `conda-libmamba-solver` plugin[^4] wired in libmamba's libsolv-based solver; conda 23.10.0 (October 2023) made **libmamba the default solver**[^5], cutting typical solve times by an order of magnitude. The change is mostly invisible to users but is the single most important thing to know about modern conda performance.

Other internals worth understanding:

- **Channels and priority.** Packages are fetched from ordered channels. `strict` channel priority (now the recommended default) prevents conda from mixing builds of the same package across channels — the historical source of subtly broken environments where a package from `defaults` linked against a library from `conda-forge`.
- **`noarch` packages.** Pure-Python or data-only packages are published once as `noarch` rather than per-platform, reducing channel size and build matrix.
- **Activation.** `conda activate` is a shell function (installed via `conda init` into your shell rc file), not a plain binary. It manipulates `PATH` and runs per-package activation scripts. This shell integration is a frequent source of CI and non-interactive-shell breakage.
- **Plugin system.** Modern conda exposes a `pluggy`-based plugin API (solvers, subcommands, virtual packages, auth handlers), which is how libmamba is delivered rather than being hard-coded.

Versioning switched to CalVer (`YY.MM.MICRO`, e.g. `24.5.0`) in 2022[^6], replacing the earlier `4.x` semver line.

## Production Notes

- **Anaconda channel terms are a legal footgun, not a technical one.** The `defaults` channel and `repo.anaconda.com` are governed by Anaconda's Terms of Service, which require a paid license for use by larger organizations. Miniconda points at these channels by default. Many teams discovered this only after adoption. If you want to avoid it, use Miniforge and the `conda-forge` channel, and audit your `.condarc` for implicit `defaults`[^2].
- **Do not install into `base`.** The `base` environment holds conda itself; polluting it with project packages leads to unsolvable upgrades of conda later. Create a named environment per project.
- **Mixing `pip` and `conda` is supported but order-sensitive.** Install everything you can via conda first, then `pip install` the remainder into the same environment. Running conda operations *after* pip can leave conda unaware of pip-installed files; conda tracks what it installed, not what pip did.
- **Reproducibility needs lockfiles.** `environment.yml` records loose constraints, not a solved set — re-solving later can yield different versions. Use `conda env export --no-builds` for portability, or `conda-lock` / pixi for true cross-platform locked environments.
- **Upgrading across large version gaps can require stepping stones.** Going from a very old conda (e.g. 4.12) to a current release may need an intermediate install (e.g. `conda install -n base conda=22.11.1` first) rather than a single jump[^7].
- **CI activation.** In non-interactive shells, `conda activate` fails unless the shell was `conda init`-ed or you `source` the conda profile script. On CI, prefer `conda run -n env <cmd>` or the setup-miniconda action over relying on an activated shell.
- **Disk and speed.** Hard-linking keeps environments cheap on a single filesystem, but crossing filesystem boundaries forces copies. The package cache (`pkgs/`) grows unbounded; `conda clean --all` reclaims it.

## When to Use / When Not

**Use when:**
- You need native, non-Python dependencies (CUDA, MKL, GDAL, compilers, R) pinned and reproducible alongside Python.
- You are in data science / ML / scientific computing where prebuilt binary wheels don't cover the full stack.
- You want isolated environments that include the interpreter itself, not just libraries.

**Avoid when:**
- Your project is pure Python with good wheel coverage — pip + venv, or uv, is lighter and faster.
- You are building a slim production container and want minimal image size — conda environments are large; multi-stage builds or pip are usually leaner.
- You need the newest package versions immediately — conda channels can lag PyPI, and rebuilding for the conda ecosystem takes time.
- Your organization must avoid the Anaconda `defaults` terms and you are unwilling to standardize on `conda-forge`/Miniforge.

## Alternatives

- mamba-org/mamba — near-drop-in C++ reimplementation of the conda CLI; use it when you want conda semantics with faster solves and don't want to wait on the Python CLI (the two have largely converged since libmamba became conda's default solver).
- prefix-dev/pixi — modern Rust-based project/environment manager built on the conda ecosystem with built-in lockfiles; use it when you want a `cargo`-style per-project workflow over conda packages.
- conda-forge/miniforge — not a competitor but the recommended installer; use it instead of Miniconda when you want conda-forge defaults and to sidestep Anaconda's channel terms.
- astral-sh/uv — extremely fast Python-only package/resolver; use it when your dependencies are pure-Python wheels and you don't need non-Python binaries.
- pypa/pip + python venv — use when the project is standard Python and you want the lowest-overhead, most universally understood tooling.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2012 | Initial release by Continuum Analytics[^1]. |
| 4.7 | 2019-05 | `.conda` (zstd) package format introduced[^3]. |
| 4.6 | 2019 | `conda activate` shell-function model, strict channel priority option. |
| 22.9.0 | 2022-09 | Switch to CalVer versioning[^6]. |
| 22.11 | 2022-11 | `conda-libmamba-solver` shipped as opt-in experimental solver[^4]. |
| 23.10.0 | 2023-10 | libmamba becomes the **default** solver[^5]. |

## References

[^1]: conda project history / "Conda: myths and misconceptions" — Jake VanderPlas, 2016. https://jakevdp.github.io/blog/2016/08/25/conda-myths-and-misconceptions/
[^2]: Anaconda Terms of Service and channel licensing. https://www.anaconda.com/pricing/terms-of-service-faqs
[^3]: conda 4.7.0 release / `.conda` format announcement. https://www.anaconda.com/blog/how-we-made-conda-faster-4-7
[^4]: conda-libmamba-solver project. https://github.com/conda/conda-libmamba-solver
[^5]: "conda 23.10.0 now uses the libmamba solver by default." conda changelog. https://docs.conda.io/projects/conda/en/latest/release-notes.html
[^6]: conda CalVer versioning (`YY.MM.MICRO`). https://calver.org
[^7]: conda README — staged updates across large version gaps. https://github.com/conda/conda#updating-conda

## Tags

python, package-manager, environment-manager, dependency-resolution, scientific-computing, machine-learning, cross-platform, conda-forge, binary-packages, cli, calver
