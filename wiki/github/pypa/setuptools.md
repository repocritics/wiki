# pypa/setuptools

> The legacy-bearing build backend that most of Python still installs through — the successor to distutils and the reason `setup.py` refuses to die.

[GitHub repo](https://github.com/pypa/setuptools) ·
[Official website](https://setuptools.pypa.io) ·
[License: MIT](https://github.com/pypa/setuptools/blob/main/LICENSE)

## Overview

Setuptools is the oldest still-dominant packaging library in Python. It began around 2004 as Phillip Eby's extension of the standard library's `distutils`, adding dependency declarations, entry points, and the `easy_install` fetcher[^1]. It absorbed the `distribute` fork in 2013 (setuptools 0.7), and when `distutils` was deprecated by PEP 632 and removed from the standard library in Python 3.12, setuptools took over its role by vendoring a copy of it[^2]. For a large fraction of the ecosystem, "building a Python package" still means "running setuptools," whether or not the author knows it.

The defining tension is legacy versus standards. Setuptools predates every modern packaging PEP, so it carries imperative `setup.py`, eggs, `pkg_resources`, `easy_install`, and namespace-package machinery that the community has spent a decade trying to leave behind. At the same time it has been retrofitted to speak the new standards: it is a PEP 517 build backend (`setuptools.build_meta`), it reads declarative metadata from `setup.cfg` and, since setuptools 61 (2022), from the `[project]` table of `pyproject.toml` per PEP 621[^3], and it supports PEP 660 editable installs. The result is a tool that can be configured three different ways, all valid, with overlapping and occasionally conflicting semantics.

It is important to be precise about what setuptools is *not*: it is a build backend and metadata library, not a project manager. It does not resolve dependency graphs, produce lockfiles, or manage virtual environments. `pip` (installer/resolver) and tools like `uv`, `poetry`, and `hatch` sit above it; setuptools' job ends at producing an sdist and a wheel.

## Getting Started

Setuptools ships with most Python installations and with pip, so it is rarely installed directly. For a new project the modern path is a `pyproject.toml`:

```toml
# pyproject.toml
[build-system]
requires = ["setuptools>=61"]
build-backend = "setuptools.build_meta"

[project]
name = "my-package"
version = "0.1.0"
description = "An example package"
requires-python = ">=3.9"
dependencies = ["requests>=2.28"]

[project.scripts]
my-cli = "my_package.cli:main"
```

```bash
python -m build          # produces dist/*.tar.gz (sdist) and dist/*.whl (wheel)
pip install -e .         # editable install for development
```

With that layout there is no `setup.py` at all — setuptools discovers packages, reads metadata, and builds the wheel from the declarative config.

## Architecture / How It Works

Setuptools has three configuration surfaces that co-exist:

1. **`setup.py`** — imperative. Calls `setuptools.setup(**kwargs)`. Still required for anything dynamic (C extensions, computed metadata, custom build steps). Invoking it directly (`python setup.py install`, `python setup.py develop`) is deprecated; the supported entry point is now the PEP 517 backend[^4].
2. **`setup.cfg`** — declarative INI. The 2016-era answer to `setup.py`'s imperativeness. Still supported, now largely superseded by `pyproject.toml`.
3. **`pyproject.toml [project]`** — declarative TOML, PEP 621. The current recommended surface[^3].

Under PEP 517, a front end (`pip`, `build`) creates an isolated environment, installs `build-system.requires`, imports the declared `build-backend`, and calls `build_wheel` / `build_sdist` hooks. `setuptools.build_meta` implements those hooks and drives the rest of the machinery.

Key internal pieces:

- **Package discovery** — `find_packages()` / `find_namespace_packages()` walk the source tree. Since setuptools 61, automatic discovery can infer packages with no explicit configuration, which is convenient but silently guesses wrong for non-standard layouts.
- **`pkg_resources`** — a runtime library bundled with setuptools for locating installed distributions, entry points, and resources. It is deprecated in favor of `importlib.metadata` / `importlib.resources`, and its import is notoriously slow because it scans `sys.path` at import time.
- **Entry points** — the mechanism behind console scripts and plugin systems. Written into wheel metadata; read back via `importlib.metadata.entry_points()`.
- **C/C++ extensions** — `Extension` objects compiled through the vendored `distutils` compiler abstraction. This is the part with no modern replacement inside setuptools and the reason `setup.py` survives.
- **`MANIFEST.in` / `package_data`** — control which files land in the sdist and wheel respectively; the two are separate systems and a frequent source of "it works locally but the wheel is missing a file" bugs.

Setuptools also vendors many of its own dependencies and auto-syncs its repo scaffolding from the maintainer's "skeleton" project[^5], which keeps tooling uniform but produces a high volume of mechanical commits and occasional churn in behavior between minor releases.

## Production Notes

**Pin your build backend.** `build-system.requires = ["setuptools"]` with no ceiling means CI can pick up a new major setuptools on any given day. Setuptools releases very frequently and bumps its major version often; breaking changes (removed deprecated features, discovery changes) have repeatedly broken builds that did not pin. Pin a lower *and* consider an upper bound for reproducibility.

**`setup.py install` / `develop` are deprecated.** Use `pip install .` and `pip install -e .`. Direct `setup.py` invocations bypass build isolation and PEP 517 and emit deprecation warnings; they are slated for removal.

**`pkg_resources` import cost.** If your package or a dependency does `import pkg_resources` at startup, you pay a `sys.path` scan on every process launch — measurable in CLIs and serverless cold starts. Migrate runtime code to `importlib.metadata`.

**Editable installs changed semantics.** PEP 660 editable installs (setuptools 64+) can use an import hook or a `.pth` file depending on layout and configuration. Behavior differs from the old `develop`/egg-link approach; packages relying on `__file__` paths or auto-discovery of newly added modules sometimes see files that are not importable until reinstall.

**Namespace packages are a minefield.** There are three historical mechanisms — `pkg_resources`-style, `pkgutil`-style, and PEP 420 native — and mixing them across distributions that share a namespace produces import failures that only appear when both are installed together. Prefer PEP 420 (implicit) namespaces and `find_namespace_packages`.

**Eggs and `easy_install` are legacy.** The wheel format (PEP 427) and `pip` are the supported install path. Do not build or distribute `.egg` files for new projects; `easy_install` is deprecated and should not be used.

**Vendored distutils.** Setuptools ships its own `distutils`; the `SETUPTOOLS_USE_DISTUTILS` environment variable historically toggled between the vendored copy and the stdlib one. On Python 3.12+ the stdlib copy is gone, so the setuptools copy is authoritative — relevant if you had downstream patches against the system `distutils`.

## When to Use / When Not

**Use when:**
- You build C/C++/Cython extension modules — setuptools is the mature, best-supported backend for compiled code.
- You maintain an existing project already on setuptools and have no reason to migrate.
- You need imperative build logic (computed version, generated files, custom commands) that declarative backends do not express.
- You want the most universally understood, most widely installed backend for maximum compatibility.

**Avoid when:**
- You are starting a pure-Python package and want a clean declarative experience — a lighter PEP 517 backend has less legacy surface.
- You want an integrated project manager (env, lockfile, resolver, publish) — setuptools is only the build step.
- You need reproducible, minimal build dependencies — setuptools' vendored bulk and fast-moving releases work against that.

## Alternatives

- pypa/hatch — Hatchling backend + project manager; clean PEP 621-native, no `setup.py`, good default for new pure-Python packages.
- python-poetry/poetry — opinionated all-in-one (deps, lockfile, publish) with its own backend; use when you want managed environments over a bare build step.
- pypa/flit — minimal backend for simple pure-Python packages; use when your package has no build step at all.
- pdm-project/pdm — PEP 621 project manager with PEP 582 support; use when you want standards-first workflow management.
- PyO3/maturin — for building Rust extension modules into wheels; use instead of setuptools when the native code is Rust rather than C.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2004 | setuptools + `easy_install` created by Phillip Eby, extending distutils[^1]. |
| — | 2008 | `distribute` fork created to unblock stalled maintenance. |
| 0.7 | 2013 | `distribute` merged back into setuptools; single project again. |
| 30.3 | 2016-12 | Declarative metadata via `setup.cfg`. |
| — | 2017–18 | PEP 517/518 build backend (`setuptools.build_meta`). |
| 61.0 | 2022-03 | PEP 621 `[project]` table in `pyproject.toml`; automatic discovery[^3]. |
| 64.0 | 2022-08 | PEP 660 editable installs. |
| — | 2023-10 | Python 3.12 removes stdlib `distutils` (PEP 632); setuptools vendors it[^2]. |

## References

[^1]: Setuptools history and its relationship to distutils. https://setuptools.pypa.io/en/latest/history.html
[^2]: PEP 632 — Deprecate distutils module (removed in Python 3.12). https://peps.python.org/pep-0632/
[^3]: Setuptools support for `pyproject.toml` configuration (PEP 621), added in 61.0. https://setuptools.pypa.io/en/latest/userguide/pyproject_config.html
[^4]: "Why you shouldn't invoke setup.py directly." https://blog.ganssle.io/articles/2021/10/setup-py-deprecated.html
[^5]: jaraco's "skeleton" project scaffolding used across pypa/jaraco repos. https://blog.jaraco.com/skeleton/

## Tags

python, packaging, build-backend, setup-py, pyproject-toml, pep-517, pep-621, distutils, wheel, pypa, build-system
