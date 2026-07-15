# tox-dev/tox

> Command-line tool that builds your package once, then runs your test commands inside isolated per-environment virtualenvs — the standard way to test a Python library across many interpreter versions.

[GitHub repo](https://github.com/tox-dev/tox) ·
[Official website](https://tox.wiki) ·
[License: MIT](https://github.com/tox-dev/tox/blob/main/LICENSE)

## Overview

tox is a generic virtualenv-management and test-runner front end for Python[^1]. Its core job is a loop: for each declared environment, create a fresh virtualenv, build and install your package into it, install the test dependencies, and run your commands. Because the package is built and installed the same way an end user would install it, tox catches packaging bugs (missing `MANIFEST.in` entries, broken `pyproject.toml` metadata, import-path mistakes) that a bare `pytest` run in the source tree silently hides. It was created by Holger Krekel, who also started pytest, and the two tools grew up together.

The historical use case is the interpreter matrix: run the same suite under py39/py310/py311/py312/pypy without hand-managing virtualenvs. tox pairs naturally with CI — `tox-gh-actions` maps a GitHub Actions matrix job to a tox environment — and for years it was the default answer to "how do I test my library across Python versions." It is a developer/CI tool, not a runtime dependency: it never ships inside your application.

The defining tension is **declarative config vs. imperative scripting**. tox environments are described in a static config file (`tox.ini`, `pyproject.toml`, or `setup.cfg`) with its own substitution and factor mini-language. This is terse and reproducible but hits a ceiling when you need real conditionals or loops — the exact gap that its main competitor, nox, fills by using a plain Python file. The second tension is the tox 3 → tox 4 rewrite (2022): the entire engine was rebuilt, which fixed long-standing issues but broke plugins and subtle config behaviors that projects had depended on for a decade[^2].

## Getting Started

```bash
pipx install tox        # isolated install; also: uv tool install tox, or pip install tox
```

A `tox.ini` for a library tested on three interpreters plus a lint environment:

```ini
[tox]
env_list = py311, py312, py313, lint

[testenv]
deps =
    pytest
    pytest-cov
commands = pytest {posargs}

[testenv:lint]
skip_install = true
deps = ruff
commands = ruff check .
```

```bash
tox              # run every env in env_list
tox -e py312     # run one env
tox -p           # run envs in parallel
tox -e py312 -- -k test_login   # args after -- flow into {posargs}
```

## Architecture / How It Works

Each `[testenv]` (and `[testenv:NAME]`) section defines one environment. A run of that environment executes, in order:

1. **Provision the virtualenv** — created via the `virtualenv` library (also a tox-dev project), not the stdlib `venv`, for speed and consistency.
2. **Build the package** — tox builds an sdist/wheel through the PEP 517 build backend declared in your `pyproject.toml` (`build-backend`), so the install path matches real-world usage. `skip_install = true` opts out for lint/docs environments that don't need your code installed.
3. **Install** — the built artifact plus `deps` are installed with pip into the env.
4. **Run `commands`** — each line is a separate subprocess; a non-zero exit fails the environment.

**Factors and substitutions** are the config language. An env name like `py311-django42` splits into factors (`py311`, `django42`); dependency and command lines can be gated with `factor: value` syntax so one `[testenv]` block expands into a matrix. Substitutions like `{posargs}`, `{envname}`, `{toxinidir}`, `{env:VAR}` are expanded at runtime. This is powerful but is a bespoke DSL you have to learn, and error messages for a malformed substitution are not always obvious.

**Environment isolation** is deliberate: by default the subprocess environment is scrubbed, and only variables named in `passenv` are passed through (`setenv` sets new ones). This prevents "works on my machine" leakage but is a frequent first surprise — a test that reads `HOME`, `HTTP_PROXY`, or a CI token needs it explicitly whitelisted.

**Plugins** use the `pluggy` hook system (the same one pytest uses). The ecosystem includes `tox-gh-actions`, `tox-uv` (swap virtualenv+pip for the uv resolver/installer), `tox-docker`, and `tox-conda`. The plugin hook API changed substantially in tox 4, so plugins are version-gated.

## Production Notes

**The tox 3 → 4 migration is the big one.** tox 4 (Dec 2022) is a ground-up rewrite. Most simple `tox.ini` files run unchanged, but real projects hit differences: stricter config parsing, changed `{posargs}` and factor-expansion edge cases, altered `install_command`/`list_dependencies` behavior, and plugins that simply didn't work until updated. If you pin `tox<4` to avoid this, you're on an unmaintained line. Budget time and read the migration notes before upgrading a CI pipeline that many contributors depend on.

**Speed.** The cost model is dominated by environment creation and dependency installation, not test execution. For an N-interpreter matrix you pay N package builds and N installs. Mitigations: `tox -p` (parallel), `--develop`/`usedevelop` for an editable install during local iteration (trades packaging fidelity for speed), and — the big lever — the `tox-uv` plugin, which replaces pip+virtualenv with uv and cuts cold-run install time dramatically. Recreation happens when config or deps change; `-r`/`--recreate` forces it, and stale envs after a dependency bump are a classic "why is CI green locally but red in CI" cause.

**Config location.** `tox.ini` is canonical. Native TOML config under `[tool.tox]` in `pyproject.toml` is supported in recent tox 4 releases[^3]; before that, projects embedded the ini blob as a string via `legacy_tox_ini`. If you want a single-file project, check your tox version supports native TOML before deleting `tox.ini`.

**CI reality.** tox predates rich CI matrix support. Today the choice is "GitHub Actions matrix runs one interpreter per job" vs. "one job runs `tox` over all interpreters." The former parallelizes across runners and gives clearer logs; `tox-gh-actions` bridges the two by selecting the right env from the job's Python version. Running the full matrix inside one job is slower and harder to read, but portable across CI providers.

**Footguns.** Environment scrubbing surprising a test (see `passenv`); `commands` that need shell features (pipes, `&&`) don't get a shell — each line is `exec`'d directly, so wrap in `bash -c` or split the steps; and `allowlist_externals` is required for any command not installed into the env (git, make, docker), which trips up first-time users.

## When to Use / When Not

**Use when:**
- You maintain a library that must work across several Python versions and want packaging correctness verified on every run.
- You want reproducible, declarative test environments that a contributor can run with one command.
- You're standardizing CI across providers and want the test definition to live in the repo, not in a provider's YAML.
- You already know the tox config DSL and your matrix is expressible in factors.

**Avoid when:**
- Your automation needs real logic (conditionals, dynamic env generation, shared Python helpers) — nox's Python config is a better fit.
- You're an application (not a library) with one target interpreter and no packaging step — a plain virtualenv + pytest or a `Makefile`/`justfile` is simpler.
- You've adopted a tool that already owns environments end to end (Hatch, PDM, uv) and don't want a second environment manager.
- You need shell-heavy task orchestration — tox's `commands` model fights you.

## Alternatives

- wntrblm/nox — same problem, but sessions are defined in a `noxfile.py` (real Python). Use nox when your matrix needs conditionals, loops, or shared helper code the ini DSL can't express.
- pypa/hatch — packaging tool with a built-in environment/matrix manager. Use Hatch when you want one tool for build, publish, and test environments rather than tox layered on top.
- astral-sh/uv — extremely fast installer/resolver that can run scripts and manage envs. Use uv (or `tox-uv`) when install speed dominates; uv alone can replace tox for simple single-interpreter flows.
- pdm-project/pdm — dependency manager whose scripts/environments overlap tox for project-local task running. Use PDM when it's already your dependency manager.
- Bare pytest + a `Makefile`/`justfile` — use when there's a single interpreter and no packaging matrix to justify tox's machinery.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2011 | Initial release by Holger Krekel; per-env virtualenv test loop[^1]. |
| 2.0 | 2015 | Environment-variable isolation (`passenv`/scrubbed subprocess env). |
| 3.0 | 2018 | Config and internals overhaul; long-stable tox 3 line. |
| 4.0 | 2022-12 | Full rewrite: new engine, revamped plugin API, stricter config[^2]. |
| 4.x | 2023–2026 | `tox-uv` ecosystem, native `[tool.tox]` TOML config in `pyproject.toml`[^3]. |

## References

[^1]: tox documentation — "tox aims to automate and standardize testing in Python." https://tox.wiki/en/latest/
[^2]: tox changelog / "tox 4 — the plugin ecosystem and migration." https://tox.wiki/en/latest/changelog.html
[^3]: tox configuration reference — `pyproject.toml` / `[tool.tox]` support. https://tox.wiki/en/latest/config.html

## Tags

python, testing, automation, virtualenv, ci, test-runner, cli, packaging, developer-tools, matrix-testing
