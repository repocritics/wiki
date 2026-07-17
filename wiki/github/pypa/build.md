# pypa/build

> A PEP 517 build frontend: the tool that turns a source tree into an sdist and a wheel, without caring which backend does the work.

[GitHub repo](https://github.com/pypa/build) ·
[Official website](https://build.pypa.io) ·
[License: MIT](https://github.com/pypa/build/blob/main/LICENSE)

## Overview

`build` is the reference build *frontend* maintained by the Python Packaging Authority (PyPA). Its single job is to take a project directory, read the `[build-system]` table in `pyproject.toml`, provision an isolated environment with the declared build dependencies, and invoke the backend's PEP 517 hooks to produce distribution artifacts in `dist/`[^1]. It is the standards-compliant replacement for the historical `python setup.py sdist bdist_wheel` invocation, which hard-coded setuptools as the only possible backend.

The defining design choice is *frontend/backend separation*, formalized in PEP 517 and PEP 518[^2]. `build` knows nothing about how a wheel is actually assembled — that is the backend's problem (setuptools, hatchling, flit-core, poetry-core, pdm-backend, maturin, scikit-build-core, and others). This is why the project's own tagline is "a simple, correct Python build frontend": it deliberately does very little, and treats "correct" (spec-conformant, reproducible, no accidental reliance on the ambient environment) as the feature rather than speed or breadth.

The tension worth naming up front: `build` is intentionally minimal and slow-ish, because build isolation means creating a fresh virtual environment and installing build requirements on nearly every run. For a fast inner loop developers increasingly reach for `uv build` or a backend's own CLI; `build` remains the neutral, backend-agnostic tool you standardize on in CI and release automation.

## Getting Started

```console
$ pip install build
$ python -m build
```

By default this builds an sdist from the source tree, then builds a wheel *from that sdist* — catching the common mistake of shipping a wheel that only works because of files present in your working directory but missing from the sdist. Both land in `dist/`.

```console
# Just a wheel, built directly from the source tree
$ python -m build --wheel

# No isolation (use the current environment's already-installed build deps)
$ python -m build --no-isolation --skip-dependency-check

# Speed up env creation by using uv as the installer
$ pipx run 'build[uv]' --installer uv

# Pass a PEP 517 config setting through to the backend
$ python -m build -C--build-option=--verbose
```

The installed console script is `pyproject-build`; `python -m build` and `pyproject-build` are equivalent entry points.

## Architecture / How It Works

A run of `build` is a small orchestration loop around the PEP 517 hook protocol:

1. **Parse `pyproject.toml`.** Read `[build-system] requires` and `build-backend`. If the table is absent, `build` falls back to the legacy default of `setuptools` + `wheel` per PEP 518.
2. **Create an isolated environment.** By default a throwaway venv (or `virtualenv`) is created and the `requires` are installed into it. Isolation is what makes the build reproducible independent of whatever happens to be in your active environment.
3. **Query dynamic requirements.** Call the backend's `get_requires_for_build_sdist` / `get_requires_for_build_wheel` hooks and install any additional dependencies they report.
4. **Invoke the build hook.** Call `build_sdist` and/or `build_wheel` (and optionally `prepare_metadata_for_build_wheel` for the `--metadata` path).

The hook invocation itself is delegated to the `pyproject_hooks` library (formerly `pep517`), which runs each backend hook in a *subprocess* against an in-tree caller shim. `build` proper is a thin coordinator: environment lifecycle, argument parsing, and the sdist→wheel default flow. Its runtime dependencies are deliberately few — `packaging`, `pyproject_hooks`, and (on older interpreters) `tomli` and `importlib.metadata` backports.

The `--installer` flag selects how build requirements are installed into the isolated environment: `pip` (default) or `uv`. `uv` is dramatically faster at resolving and installing, which matters because environment creation dominates wall-clock time for small projects. `build` does not itself resolve or lock anything — it defers entirely to the chosen installer.

Because the backend runs in isolation, a project's build is only as reproducible as its `[build-system] requires` pins. `build` will faithfully reproduce whatever the backend does; it does not add its own layer of dependency locking.

## Production Notes

- **Isolation cost is real.** Every default run creates a virtual environment and installs build deps from scratch. In CI matrices this adds up. Mitigations: `--installer uv` (much faster installs), or `--no-isolation` when you have already provisioned the exact build deps and pass `--skip-dependency-check` (`-nx`) to skip verification. `-nx` matches pip/uv's `--no-build-isolation` semantics.
- **`--no-isolation` is a footgun if misused.** It builds against your *current* environment, so a missing or mismatched build dependency yields a wheel that "works on my machine." Reserve it for controlled environments (distro packaging, conda-forge, air-gapped builds) where deps are pinned externally.
- **The default builds a wheel from the sdist, not the source tree.** This is a feature: it validates that your `MANIFEST.in` / sdist inclusion rules actually capture everything the wheel needs. But it means `python -m build` runs the backend twice and is slower than `--wheel`. If you only need a wheel for local testing, use `--wheel` to build directly from the tree.
- **Config settings are backend-specific and inconsistently supported.** The `-C` / `--config-json` PEP 517 `config_settings` mechanism is a passthrough; what a given key does depends entirely on the backend. Setuptools' support is limited, and the README says so explicitly.
- **Network access during builds.** Isolation implies installing build requirements, which normally means hitting PyPI. Offline or hermetic builds need a local index or `--no-isolation` with pre-installed deps.
- **It is a frontend, not an installer or a project manager.** `build` produces artifacts; it does not install your package, manage lockfiles, or run tests. Pairing it with `twine` (upload) is the classic release flow; `uv`/`hatch`/`pdm`/`poetry` cover the fuller project-lifecycle role.

## When to Use / When Not

**Use when:**
- You need standards-compliant sdist + wheel artifacts and want to be backend-agnostic.
- You are writing CI/release automation and want a stable, well-scoped tool (`cibuildwheel` uses it as the default build frontend in 3.0+).
- You want the sdist→wheel round-trip validation that the default flow gives for free.
- You maintain a library and don't want to commit to one backend's bespoke CLI.

**Avoid (or look elsewhere) when:**
- You want a fast inner development loop — `uv build` or the backend's own command will be quicker.
- You need dependency resolution, locking, virtualenv management, or publishing in one tool — that is `uv`/`hatch`/`pdm`/`poetry` territory.
- You are only ever building with one backend and are happy calling it directly (e.g. `flit build`).
- Your build must be fully offline without pre-provisioning build deps.

## Alternatives

- astral-sh/uv — `uv build` produces sdists/wheels far faster; reach for it when build speed and an integrated toolchain matter more than backend neutrality.
- pypa/hatch — use when you want a full project manager (envs, versioning, scripts) whose `hatchling` backend it drives.
- pypa/setuptools — the backend itself; call it directly only for legacy `setup.py`-driven builds you can't migrate.
- pdm-project/pdm — use when you want PEP 582/lockfile-centric workflow with an integrated build step.
- pypa/cibuildwheel — use for producing many platform-specific binary wheels in CI; it wraps `build` rather than replacing it.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.0.1 | 2020-05-25 | First PyPI release; PEP 517 frontend proof of concept[^3]. |
| 0.1.0 | 2020-10-29 | Early public API; sdist→wheel default flow. |
| 0.7.0 | 2021-09-16 | Maturing config-settings and isolation handling. |
| 1.0.0 | 2023-09-01 | First stable release; API committed[^4]. |
| 1.1.0 | 2024-02-29 | `pyproject_hooks` integration and packaging updates. |
| 1.2.0 | 2024-03-27 | `--installer` selection (pip/uv), `[uv]` extra. |
| 1.3.0 | 2025-08-01 | Continued maintenance and Python version support. |
| 1.4.0 | 2026-01-08 | `--config-json` complex config settings. |
| 1.5.1 | 2026-07-09 | Latest release as of writing. |

## References

[^1]: Project README and documentation — "A simple, correct Python build frontend." https://build.pypa.io
[^2]: PEP 517 (build backend interface) and PEP 518 (`pyproject.toml` build-system table). https://peps.python.org/pep-0517/ and https://peps.python.org/pep-0518/
[^3]: PyPI release history for `build`. https://pypi.org/project/build/#history
[^4]: `build` 1.0.0 changelog. https://build.pypa.io/en/stable/changelog.html

## Tags

python, packaging, build-tools, pep-517, wheel, sdist, pyproject, cli, pypa, ci
