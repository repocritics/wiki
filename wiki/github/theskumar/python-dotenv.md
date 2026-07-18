# theskumar/python-dotenv

> Reads key-value pairs from a `.env` file into `os.environ` — the Python
> ecosystem's default answer to 12-factor configuration in development.

[GitHub repo](https://github.com/theskumar/python-dotenv) ·
[Official website](https://saurabh-kumar.com/python-dotenv/) ·
[License: BSD-3-Clause](https://github.com/theskumar/python-dotenv/blob/main/LICENSE)

## Overview

python-dotenv loads environment variables from a `.env` file, following the
convention popularized by Ruby's dotenv and codified by the 12-factor app
methodology: configuration lives in the environment, and a local file stands in
for that environment during development[^1]. The project dates to 2014 and is
maintained by Saurabh Kumar and Bertrand Bonnefoy-Claudet[^2]. Its ~8.8k GitHub
stars understate its reach: it is a transitive dependency of much of the Python
web ecosystem (Flask's CLI integrates it directly, uvicorn's `--env-file` uses
it) and sits among the most-downloaded packages on PyPI[^3].

The defining tension is scope. python-dotenv deliberately does one thing — parse
a file, put strings in the environment — and refuses to grow into a
configuration framework. There is no type casting, no validation, no schema, no
secrets backend. Every value is a `str` or `None`. Teams that outgrow this
either layer a typed settings library on top (pydantic-settings, environs) or
migrate away entirely. The second tension is that the `.env` format itself has
no formal specification; python-dotenv's parser is "similar to Bash" but
diverges from Bash and from other dotenv implementations (Node, Ruby,
docker-compose) in escaping, quoting, and interpolation details, and the
maintainers state the format "still improves over time"[^1].

## Getting Started

```bash
pip install python-dotenv
```

```python
# .env in the project root:
#   DOMAIN=example.org
#   ADMIN_EMAIL=admin@${DOMAIN}

from dotenv import load_dotenv, dotenv_values

load_dotenv()          # finds .env, sets os.environ; existing vars win
                       # (pass override=True to reverse that)

config = dotenv_values(".env")   # dict, does NOT touch os.environ
```

A CLI ships as an extra (`pip install "python-dotenv[cli]"`): `dotenv set`,
`dotenv list`, and `dotenv run -- python app.py` to launch a subprocess with the
file's variables applied[^1].

## Architecture / How It Works

The package is small, pure Python, and dependency-free (the CLI extra pulls in
click). The interesting parts:

- **Parser** (`parser.py`) — line-oriented, regex-driven. Handles unquoted,
  single- and double-quoted values, `export` prefixes, inline comments,
  multiline quoted values, and keys with no value (which parse to `None` in
  `dotenv_values` but are skipped by `load_dotenv`). Malformed lines are
  reported and skipped rather than aborting the whole file.
- **Interpolation** (`variables.py`) — POSIX-style `${VAR}` and
  `${VAR:-default}` expansion. Bare `$VAR` is *not* expanded — a recurring
  surprise for people sharing `.env` files with docker-compose or shell
  scripts. Resolution order depends on `override`: with `override=True` the
  file wins over the environment; with the default `override=False` the
  environment wins[^1].
- **Discovery** (`find_dotenv`) — walks up the directory tree looking for
  `.env`. To pick a starting point it inspects the call stack to locate the
  caller's file; in REPLs or frozen executables it falls back to the current
  working directory (`usecwd=True` forces this). This stack-frame heuristic is
  the source of most "works locally, not under gunicorn/PyInstaller" reports.
- **Write path** — `set_key` / `unset_key` rewrite the `.env` file in place,
  which is what the CLI uses. An IPython extension (`%dotenv`) is included.

Since the 1.2 series, setting `PYTHON_DOTENV_DISABLED=1` turns `load_dotenv()`
into a no-op — an escape hatch for production environments where a third-party
package calls it on import and you cannot remove the call[^4].

## Production Notes

- **Silent no-op is the top footgun.** If no `.env` file is found,
  `load_dotenv()` returns `False` and does nothing. Deploys that accidentally
  rely on a `.env` that is not shipped fail late, with missing-variable errors
  far from the cause. Check the return value or assert required keys at
  startup.
- **`override=False` is the default.** Already-exported environment variables
  shadow the file. This is correct 12-factor behavior (real environment beats
  dev file) but regularly confuses developers editing `.env` while a stale
  export lingers in their shell.
- **Load order matters.** `load_dotenv()` mutates `os.environ`; any module that
  read a variable at import time before the call sees the old value. Call it
  as the first thing in your entrypoint, not from a deeply imported module.
- **Multiple loaders interact.** Flask auto-loads `.env`/`.flaskenv` when
  python-dotenv is installed[^5], uvicorn has `--env-file`, pydantic-settings
  parses `.env` itself, and docker-compose interpolates the same file with
  different rules. Stacking these produces precedence puzzles; pick one loader
  per process.
- **Not a secrets manager.** `.env` files are plaintext on disk and routinely
  leak via image layers, backups, and accidental commits. The project's own
  advice is to gitignore the file[^1]; for production secrets use your
  platform's secret store and reserve python-dotenv for development.
- **No types, no validation.** `PORT=8000` arrives as the string `"8000"`.
  Casting bugs (`bool("False") is True`) are on you.
- Performance is a non-issue: one small file parsed once at startup. The
  project is conservatively maintained — releases are infrequent
  (roughly one to three per year lately) but the repo is active, with pushes
  as recent as July 2026[^6].

## When to Use / When Not

**Use when:**
- You want local development parity with environment-variable-driven config
  (12-factor) without exporting variables by hand.
- You need the de facto standard: teammates, frameworks, and tutorials assume
  `.env` + python-dotenv.
- You want layered config dicts (`dotenv_values` merged with `os.environ`)
  without touching the process environment.

**Avoid when:**
- You need typed, validated settings — use pydantic-settings or environs on
  top, or instead.
- You are managing production secrets — use a real secret store; a plaintext
  file is the wrong tool.
- Your config spans formats and environments (TOML/YAML, per-env layering,
  Vault) — dynaconf-class tools cover that.
- Your framework already loads `.env` (Flask CLI, pydantic-settings users):
  adding a second loader just adds precedence ambiguity.

## Alternatives

- pydantic/pydantic-settings — use instead when you want typed, validated
  settings classes; it reads `.env` files itself.
- sloria/environs — use instead when you want casting and validation with a
  lighter footprint than pydantic.
- HBNetwork/python-decouple — use instead if you prefer strict config/code
  separation with casting and `.ini` support.
- joke2k/django-environ — use instead in Django projects that want
  `DATABASE_URL`-style URL parsing.
- dynaconf/dynaconf — use instead for layered multi-environment config with
  TOML/YAML/JSON and secrets backends.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2014–2023 | Long pre-1.0 series; repo created 2014-09[^6]. Parser and interpolation rules evolved incrementally. |
| 0.21.1 | 2023-01-21 | Final 0.x release[^7]. |
| 1.0.0 | 2023-02-24 | First stable release after ~9 years of 0.x[^7]. |
| 1.1.0 | 2025-03-25 | Maintenance release; updated Python support matrix[^7]. |
| 1.2.0 | 2025-10-26 | `PYTHON_DOTENV_DISABLED` opt-out added around this series[^4]. |
| 1.2.2 | 2026-03-01 | Latest release as of this writing[^7]. |

## References

[^1]: python-dotenv README. https://github.com/theskumar/python-dotenv#readme
[^2]: README, Acknowledgements — maintainers Saurabh Kumar (theskumar) and Bertrand Bonnefoy-Claudet (bbc2). https://github.com/theskumar/python-dotenv#acknowledgements
[^3]: PyPI download statistics for python-dotenv. https://pypistats.org/packages/python-dotenv
[^4]: README, "Disable load_dotenv" — `PYTHON_DOTENV_DISABLED=1`. https://github.com/theskumar/python-dotenv#disable-load_dotenv
[^5]: Flask CLI documentation, "Environment Variables From dotenv". https://flask.palletsprojects.com/en/stable/cli/#environment-variables-from-dotenv
[^6]: GitHub repository metadata (fetched 2026-07): created 2014-09-06, last push 2026-07-12, 8,825 stars, 546 forks, BSD-3-Clause. https://github.com/theskumar/python-dotenv
[^7]: GitHub releases for python-dotenv. https://github.com/theskumar/python-dotenv/releases

## Tags

python, dotenv, environment-variables, configuration, 12-factor, devops, developer-tools, cli, library
