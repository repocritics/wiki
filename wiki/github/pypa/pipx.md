# pypa/pipx

> Install and run Python command-line applications, each in its own isolated virtualenv, without polluting your global environment.

[GitHub repo](https://github.com/pypa/pipx) ·
[Official website](https://pipx.pypa.io) ·
[License: MIT](https://github.com/pypa/pipx/blob/main/LICENSE)

## Overview

pipx installs Python **applications** — packages that expose command-line entry points — each into a dedicated virtual environment, then exposes their commands on your `PATH`. It is the Python analog of `npx`/`npm -g` or Homebrew: use it for tools like `black`, `ruff`, `httpie`, `poetry`, `awscli`, or `pre-commit` that you want to *run*, not *import*. It is explicitly not a replacement for `pip` when you are building an application and need libraries in your project environment[^1].

The core idea is isolation. Running `sudo pip install` into the system interpreter creates dependency conflicts and can break the OS; installing everything into one shared user environment eventually produces the same mess at smaller scale. pipx gives every tool its own venv so their dependency trees never collide, and uninstalls are clean because removing the venv removes everything. It always runs with regular user permissions and never calls `sudo pip`[^1].

pipx was created by Chad Smith in 2018 (originally under `pipxproject/pipx`) and later adopted into the Python Packaging Authority, moving to `pypa/pipx`[^2]. It is a thin orchestration layer: it does not resolve or download packages itself — it drives `pip` and the stdlib `venv` under the hood. That design keeps it small and predictable, but also means it inherits pip's resolver behavior and its failure modes.

## Getting Started

```bash
# macOS
brew install pipx
pipx ensurepath          # add pipx's bin dir to PATH (edits your shell rc)

# Linux (Debian/Ubuntu also ship `apt install pipx`)
python3 -m pip install --user pipx
python3 -m pipx ensurepath

# Windows
scoop install pipx
pipx ensurepath
```

```bash
# Install an app globally-available on your PATH
pipx install ruff
ruff --version

# Run a tool once in a temporary, cached environment (never installed)
pipx run pycowsay moo

# Add a plugin/dependency into an existing app's venv
pipx install poetry
pipx inject poetry poetry-plugin-export

# Housekeeping
pipx list
pipx upgrade-all
pipx uninstall ruff
```

After `ensurepath` you must restart your shell (or `source` your rc file) before the newly-exposed commands are found.

## Architecture / How It Works

For each `pipx install <pkg>`, pipx:

1. Creates a virtual environment under its data dir — on Linux typically `~/.local/pipx/venvs/<pkg>/`, resolved via `platformdirs` so the exact path varies by OS and by the `PIPX_HOME` env var.
2. Invokes `pip install <pkg>` inside that venv. pipx does no dependency resolution of its own; pip owns that entirely.
3. Discovers the package's console-script entry points and links them into a bin directory (`~/.local/bin` by default, `PIPX_BIN_DIR` to override). On POSIX these are symlinks; on Windows they are copied launcher shims.

`pipx run <pkg>` is a different path: it builds a temporary environment, caches it (default cache lifetime is roughly two weeks), and reuses it on subsequent runs of the same spec before rebuilding. This is what makes `pipx run` feel fast on repeat invocations and is how tools ship "no-install" quickstarts.

Key subcommands map cleanly onto this model: `inject` pip-installs extra packages into an app's existing venv (used for plugin systems like Poetry or mkdocs); `runpip` runs raw pip inside a managed venv; `reinstall` / `reinstall-all` tear down and rebuild venvs against a chosen interpreter; `--suffix` lets you install two versions of the same tool side by side (e.g. `black` and `black@old`).

Because each venv hard-references the Python interpreter it was created with (by path), the interpreter identity is baked in at install time. This single fact is the source of most real-world breakage — see below.

## Production Notes

- **Interpreter upgrades break every venv.** pipx venvs symlink to the specific `python3.x` binary they were built with. When Homebrew bumps its Python formula, or a distro upgrade removes the old minor version, all those symlinks dangle and every installed tool stops working. The fix is `pipx reinstall-all --python $(which python3.13)`, but it is a manual, surprising step and the most-reported pain point[^3]. Pin your interpreter deliberately if you care about stability.
- **PATH setup is a real onboarding cliff.** `pipx ensurepath` edits shell rc files but cannot affect the already-running shell; users routinely `pipx install X`, get "command not found", and assume it failed. In CI, prefer explicit `~/.local/bin` on `PATH` over relying on `ensurepath`.
- **`pipx run` caching can serve stale versions.** Within the cache window, `pipx run tool` reuses the cached environment rather than re-checking PyPI for a newer release. For reproducibility, pin the version in the spec (`pipx run tool==1.2.3`) rather than assuming `run` is always latest.
- **It only installs apps, not libraries.** If a package has no console entry point, `pipx install` warns that nothing was exposed. Use `pipx run --spec <pkg> <module>` or fall back to a normal venv.
- **System-wide/global installs are not the default.** pipx is user-scoped by design. A `--global` mode exists but requires writable shared directories on `PATH`; do not reach for `sudo` to get there.
- **It is a pip wrapper, with pip's constraints.** Custom index URLs, `--pip-args`, private repos, and resolver conflicts all behave as pip does. Debugging a failed install usually means reading pip's output, not pipx's.
- **Data directory portability.** Moving `~/.local/pipx` between machines or Python versions does not "just work" because of the baked-in interpreter paths; treat `pipx list` + a reinstall script as your source of truth instead.

## When to Use / When Not

**Use when:**
- You want CLI tools (linters, formatters, scaffolders, cloud CLIs) available globally but isolated from each other and from your projects.
- You need to run a tool once without committing to installing it (`pipx run`).
- You want clean, complete uninstalls and no `sudo pip` in your life.
- You maintain a plugin-based tool (Poetry, mkdocs, datasette) and need `inject` to add plugins.

**Avoid when:**
- You need libraries importable in a project — that is `pip`/`venv`/`uv` inside your project, not pipx.
- You want the fastest possible installs and a single tool for both apps and project deps — `uv tool` / `uvx` cover the same use case far faster.
- The package has no entry points, or you need tight control over the exact interpreter across upgrades without periodic `reinstall-all`.

## Alternatives

- astral-sh/uv — `uv tool install` and `uvx` do the same isolated-app job with a Rust resolver; use it instead when install/run speed matters or you already use uv for project management.
- pypa/pip — use plain `pip install` into a project venv when you need importable libraries, not standalone commands.
- Homebrew (Homebrew/brew) — use for tools that happen to be packaged in brew when you don't want a Python-specific manager and accept the maintainers' version pinning.
- mamba-org/condax — use when your tools live in the conda ecosystem and you want the same isolated-app pattern over conda envs.
- cs01/pipsi — pipx's spiritual predecessor; unmaintained, kept here only for historical context.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018 | Created by Chad Smith as `pipxproject/pipx`, successor concept to pipsi[^2]. |
| 0.15+ | 2020 | `pipx run` temporary-environment execution matured. |
| 1.0.0 | 2021-12 | First stable major release; API/CLI considered stable[^4]. |
| — | ~2022 | Repository adopted by PyPA, moved to `pypa/pipx`[^2]. |
| 1.x | 2023–2026 | `platformdirs` for paths, `--global` installs, standalone-interpreter support, ongoing pip-version compatibility work. |

## References

[^1]: pipx README and docs — "Install and Run Python Applications in Isolated Environments." https://pipx.pypa.io
[^2]: pipx project home / repository (formerly `pipxproject/pipx`, now under the Python Packaging Authority as `pypa/pipx`). https://github.com/pypa/pipx
[^3]: pipx docs, "How to reinstall all packages" (interpreter-upgrade recovery via `reinstall-all`). https://pipx.pypa.io/stable/how-to/
[^4]: pipx changelog / release history. https://pipx.pypa.io/stable/changelog/

## Tags

python, cli, packaging, virtualenv, pip, developer-tools, package-manager, pypa, isolation, dev-tooling
