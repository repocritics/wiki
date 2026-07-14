# pypa/pip

> The Python package installer — the default, bundled way to put packages from PyPI onto a Python interpreter.

[GitHub repo](https://github.com/pypa/pip) ·
[Official website](https://pip.pypa.io/) ·
[License: MIT](https://github.com/pypa/pip/blob/main/LICENSE.txt)

## Overview

pip is the reference package installer for Python: it resolves, downloads, and installs distributions from the Python Package Index (PyPI) and other indexes into a Python environment[^1]. It is maintained under the Python Packaging Authority (PyPA) and has been bundled with CPython since Python 3.4 via the `ensurepip` module (PEP 453)[^2], which is why it is the tool nearly every Python developer reaches for first. The name is a recursive acronym ("pip installs packages"); the project descends from `pip` 1.0 (2011), which itself grew out of the older `pyinstall` and eventually displaced `easy_install`/setuptools as the default installer.

pip's defining characteristic is that it is an *installer*, not a project or environment manager. It does not create virtual environments (that is `venv`/`virtualenv`), it does not write or maintain a lockfile, and it does not manage a `pyproject.toml`'s dependency list for you. `requirements.txt` is a plain input file, not a resolver-produced lock — the same file installed twice can yield different transitive versions unless every dependency is pinned. This minimalism is deliberate and is the single most common source of friction: users coming from npm, Cargo, or Bundler expect reproducible lockfiles and a resolver-of-record, and pip provides neither out of the box.

The other defining tension is the 2020 resolver migration. Until pip 20.3, pip used a naive "first-found-wins" install order that could silently produce environments with incompatible dependencies. The new backtracking resolver (default since 20.3, late 2020) actually verifies constraints — at the cost of sometimes-dramatic slowdowns and, for genuinely unsatisfiable constraint sets, long backtracking loops[^3].

## Getting Started

pip ships with Python. Verify and upgrade it:

```bash
python -m pip --version
python -m pip install --upgrade pip
```

Install packages into the current environment (use a virtual environment):

```bash
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
python -m pip install "requests>=2.31,<3"
```

Pin an environment and reinstall it elsewhere:

```bash
python -m pip freeze > requirements.txt      # snapshot installed versions
python -m pip install -r requirements.txt    # reinstall the snapshot
```

Editable install of a local project (development mode):

```bash
python -m pip install -e .
```

Prefer `python -m pip` over a bare `pip` — it guarantees you install into the interpreter you think you are, not whichever `pip` shim happens to be first on `PATH`.

## Architecture / How It Works

An `install` runs, roughly, as these stages:

1. **Requirement collection** — parse the command line, `requirements.txt`, and `constraints.txt` into requirement objects.
2. **Resolution** — the `resolvelib`-based backtracking resolver walks the dependency graph, choosing versions that satisfy all constraints simultaneously. To evaluate a candidate it needs that candidate's metadata (its own dependencies).
3. **Metadata acquisition** — historically pip downloaded a whole wheel or sdist just to read its dependencies. PEP 658/714 lets an index serve metadata separately, and PEP 691's JSON Simple API speeds discovery, so modern pip against a modern index (PyPI) can often avoid the full download[^4].
4. **Build** — for source distributions, pip invokes a build backend in an isolated environment per PEP 517/518, driven by the project's `pyproject.toml` `[build-system]` table. setuptools is the default backend when none is declared[^5].
5. **Install** — wheels (PEP 427, the standardized binary format) are unpacked directly into `site-packages`; there is no arbitrary `setup.py install` execution for wheels, which is faster and safer than the legacy path.

pip vendors its own runtime dependencies under `pip._vendor` (requests, resolvelib, packaging, and others) so that installing or upgrading a package can never break pip itself by changing a library pip depends on. This is why pip appears to have "no dependencies" — they are copied in-tree.

pip has no persistent state beyond an HTTP cache and the wheel cache under `~/.cache/pip`. It reads the environment it is pointed at (via the running interpreter's `sys.path` and `sysconfig` schemes) and writes into that environment's `site-packages`. It does not track *why* a package was installed, so it cannot cleanly uninstall a package's now-unused dependencies — there is no `pip autoremove`.

## Production Notes

**The resolver can be slow, and slowness is not a hang.** On a fresh or under-constrained install, backtracking may download and inspect many candidate versions. When constraints genuinely conflict, you can see the "This is taking longer than usual" message followed by a long loop before pip gives up. Mitigations: pin more (upper bounds help the resolver prune), pass `--constraint` files, and keep pip current — resolver performance and metadata-fetch behavior improve most releases.

**`requirements.txt` is not a lockfile.** `pip freeze` captures exact versions but not hashes, not markers cleanly, and not the resolution graph. For reproducibility, teams add hash-checking mode (`--require-hashes`, all-or-nothing) or, more commonly, use `pip-tools`' `pip-compile` to produce a fully pinned, hashed `requirements.txt` from a loose input. pip alone will not enforce a lock.

**PEP 668 / externally-managed environments.** Since ~2023, Debian/Ubuntu, Homebrew, and other distributors mark the system Python as "externally managed," and pip refuses global installs into it with an `externally-managed-environment` error[^6]. This is intentional protection against pip and the OS package manager fighting over `site-packages`. The correct fix is a virtual environment or `pipx` for applications — not `--break-system-packages`.

**Editable installs changed under the hood.** `pip install -e` now goes through PEP 660 (editable wheels) with modern backends; older setuptools `develop`/`.egg-link` behavior differed, and some tooling that scraped `.egg-link` files broke during the transition.

**Upgrade behavior is not transitive by default.** `pip install -U somepkg` upgrades that package and installs what it needs, but does not proactively upgrade already-satisfied dependencies. There is no built-in "upgrade everything" command; `pip list --outdated` plus scripting, or a higher-level tool, is the usual answer.

**No dependency-conflict *record*.** Because pip installs into a shared flat `site-packages`, two packages requiring incompatible versions of a third cannot coexist. pip will resolve to one and may print a "dependency resolver does not currently take into account all the packages that are installed" warning after the fact when you install into an already-populated environment piecemeal.

## When to Use / When Not

**Use when:**
- You want the standard, always-present tool that every Python tutorial, CI image, and Dockerfile already assumes.
- You need to install into an interpreter you control (a venv, a container, a CI job).
- You are the target of `pip install yourpackage` and need the lowest-common-denominator install path to work.
- You want a thin installer and are supplying your own reproducibility layer (pip-tools, hashes, a lockfile from another tool).

**Avoid / augment when:**
- You need fast, reproducible, lockfile-driven installs — reach for uv or Poetry/PDM instead of hand-rolling it on pip.
- You are installing *applications* (CLI tools) globally — use pipx so each gets its own isolated environment.
- Your dependency set includes heavy native/scientific stacks with non-Python system libs — conda/mamba resolve those better than PyPI wheels alone.
- You need one command to manage venv + deps + lock + publish — pip is deliberately only the install step.

## Alternatives

- astral-sh/uv — use instead when you want a single fast Rust tool that is a near drop-in for pip plus locking and venv management; the current momentum choice.
- python-poetry/poetry — use when you want an all-in-one project manager with a lockfile and its own `pyproject.toml` workflow.
- pdm-project/pdm — use when you want PEP 621 standard metadata with locking and PEP 582/venv flexibility.
- pypa/pipenv — use when you specifically want the older Pipfile/Pipfile.lock workflow; largely superseded now.
- conda/conda — use when you install cross-language native stacks (CUDA, MKL, geospatial) where PyPI wheels fall short.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2011-04 | First release under the `pip` name (formerly `pyinstall`)[^1]. |
| — | 2014-03 | Bundled with CPython 3.4 via `ensurepip` (PEP 453)[^2]. |
| 8.0 | 2016-01 | Hash-checking mode, constraints files. |
| 10.0 | 2018-04 | Build isolation (PEP 518), dropped Python 2.6, `pyproject.toml` awareness[^5]. |
| 19.0 | 2019-01 | PEP 517 build backends supported. |
| 20.3 | 2020-11 | New backtracking resolver becomes the default[^3]. |
| 21.0 | 2021-01 | Dropped Python 2.7 / 3.5 support. |
| 23.1 | 2023-04 | Legacy `setup.py`-based installs deprecated in favor of PEP 517 path. |
| 25.x | 2025 | Quarterly CalVer cadence continues; ~4 releases/year[^1]. |

## References

[^1]: pip documentation and project home. https://pip.pypa.io/
[^2]: PEP 453 — Explicit bootstrapping of pip in Python installations. https://peps.python.org/pep-0453/
[^3]: pip blog, "Changes to the pip dependency resolver in 20.3 (2020)". https://pip.pypa.io/en/stable/user_guide/#changes-to-the-pip-dependency-resolver-in-20-3-2020
[^4]: PEP 658 — Serve distribution metadata in the Simple Repository API. https://peps.python.org/pep-0658/
[^5]: PEP 518 — Specifying minimum build system requirements for Python projects. https://peps.python.org/pep-0518/
[^6]: PEP 668 — Marking Python base environments as "externally managed". https://peps.python.org/pep-0668/

## Tags

python, package-manager, packaging, pypi, dependency-resolution, installer, cli, pypa, virtualenv, developer-tools
