# pypa/twine

> The upload step for Python packaging — takes already-built artifacts and pushes them to PyPI over verified TLS.

[GitHub repo](https://github.com/pypa/twine) ·
[Official website](https://twine.readthedocs.io/) ·
[License: Apache-2.0](https://github.com/pypa/twine/blob/main/LICENSE)

## Overview

Twine is a command-line utility maintained by the Python Packaging Authority (PyPA) whose single job is uploading distribution artifacts — source distributions (`sdist`) and wheels — to PyPI or a compatible index[^1]. It deliberately does not build anything. You produce artifacts with some other tool (`build`, `setuptools`, `hatch`, `flit`, `poetry`), then hand the finished files to twine.

That separation is the whole reason twine exists. The historical `python setup.py upload` command coupled building and publishing into one step and, for years, transmitted credentials over plaintext HTTP and re-ran arbitrary `setup.py` code at upload time[^2]. Twine replaced it by uploading pre-built files over verified HTTPS to the PyPI upload API, so the artifact that gets tested locally is byte-for-byte the artifact that gets published. It is now the de facto standard publish step and the tool that most CI publishing wrappers call underneath.

The tradeoff is scope: twine is intentionally small and does one narrow thing. It is not a project manager, not a build backend, and not a version bumper. Teams that want an integrated "build, version, publish" experience increasingly reach for `hatch`, `poetry`, or `uv` — all of which either wrap or reimplement the upload step twine pioneered.

## Getting Started

```bash
pip install twine
```

```bash
# 1. Build artifacts first (twine does NOT build)
python -m build            # writes dist/*.tar.gz and dist/*.whl

# 2. Validate that the long_description renders on PyPI
twine check dist/*

# 3. Upload. Username __token__ + a PyPI API token is the standard path.
twine upload dist/*

# Upload to TestPyPI first to rehearse
twine upload --repository testpypi dist/*
```

Credentials are read, in order, from CLI flags, `TWINE_USERNAME` / `TWINE_PASSWORD` env vars, the `~/.pypirc` file, and the system keyring[^3]. For token auth, the username is the literal string `__token__` and the password is the `pypi-`-prefixed API token.

## Architecture / How It Works

Twine is a thin, focused Python package. `twine upload` does roughly: discover the files you passed, parse each artifact's metadata with the packaging machinery, prompt for or resolve credentials, then POST each file as a multipart request to the index's upload endpoint (PyPI's legacy upload API at `https://upload.pypi.org/legacy/`). A Rich-based progress bar reports transfer status.

Key dependencies define its behavior: `requests` + `requests-toolbelt` for the multipart streaming upload, `readme-renderer` for `twine check`'s render validation, `keyring` for credential storage, and `id` for retrieving OIDC tokens under Trusted Publishing. Historically twine used `pkginfo` to read artifact metadata; version 6 moved metadata parsing onto the `packaging` library, which removed a recurring class of breakage where `pkginfo` lagged behind new core-metadata versions.

`twine check` is a separate, cheaper command that only verifies the package's `long_description` will render as valid HTML on PyPI (a common silent failure — a broken README renders as raw text on the project page). It is not a full pre-flight validation: passing `check` does not guarantee the upload will succeed, and failing `check` only means the README won't render, not that the package is unusable.

Twine has no persistent state, no daemon, and no config beyond `~/.pypirc`. The coupling that matters is to PyPI's server-side rules — file-naming, metadata versions, and immutability — none of which twine controls but all of which surface as twine errors.

## Production Notes

- **Uploads are immutable.** Once a filename is published for a version, PyPI will reject re-uploading it. There is no overwrite. To fix a bad release you must bump the version (e.g. `1.2.0` → `1.2.1`); you cannot re-push `1.2.0`. Deleting a release does not free the filename for reuse.
- **`--skip-existing` for partial retries.** If an upload dies halfway (some files landed, others didn't), re-running without this flag fails on the already-uploaded files. `twine upload --skip-existing dist/*` makes the operation idempotent.
- **Password auth is effectively dead.** PyPI has disabled username/password uploads in favor of API tokens and Trusted Publishing, and requires 2FA on maintainer accounts[^4]. New setups should use an API token (`__token__`) or, in CI, Trusted Publishing.
- **In CI, prefer Trusted Publishing over stored tokens.** GitHub Actions can exchange a short-lived OIDC identity for an upload grant, so no long-lived secret sits in your repo. In practice this is done through `pypa/gh-action-pypi-publish`, which invokes twine for you; calling twine directly with a long-lived token in CI is the older, higher-risk pattern.
- **Metadata-version drift.** Because artifact metadata is validated against known core-metadata versions, a build backend emitting a newer metadata version than your installed twine understands can cause `twine check` or upload to fail. The fix is almost always upgrading twine; version 6's move to `packaging` reduced how often this happens.
- **`check` is not a gate.** Do not treat a passing `twine check` as proof the release is correct — it only tests README rendering. Wheel/sdist correctness, entry points, and dependency metadata are not validated by it.
- **File-size limits.** PyPI enforces per-file and per-project size caps (100 MB per file by default); large artifacts require a manually requested limit increase and will otherwise fail at upload, not at build.

## When to Use / When Not

**Use when:**
- You build artifacts with one tool and want a dedicated, boring, well-audited step to publish them.
- You want the artifact you tested to be exactly the artifact you ship (build/publish separation).
- You need fine control over the upload: custom repositories, `--skip-existing`, TestPyPI rehearsal, `~/.pypirc` profiles.
- You are wiring a publish step into CI and want the tool that the official GitHub Action wraps.

**Avoid / skip when:**
- Your project is already managed by `hatch`, `poetry`, or `uv` — their built-in `publish` commands cover the same ground without a separate dependency.
- You want one tool to build, version, and publish; twine is only the last step.
- You are publishing from GitHub Actions — use `pypa/gh-action-pypi-publish` (which uses twine) so you get Trusted Publishing without hand-managing tokens.

## Alternatives

- pypa/build — not a competitor but the required companion: build with `build`, then upload with twine. People conflate the two.
- pypa/gh-action-pypi-publish — use this instead of calling twine directly in GitHub Actions when you want Trusted Publishing.
- pypa/hatch — use `hatch publish` when hatch already manages your project and you want build+publish in one tool.
- python-poetry/poetry — use `poetry publish` when your project is already Poetry-managed.
- astral-sh/uv — use `uv publish` when uv is already your resolver/installer and you want a single toolchain.
- pypa/flit — use `flit publish` for a pure-Python single-module package wanting minimal configuration.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial | 2013 | Created to replace insecure `setup.py upload`; HTTPS uploads of pre-built artifacts[^2]. |
| 2.0 | 2020 | Dropped Python 2 support. |
| 3.0 | 2019 | `--non-interactive` and workflow refinements for CI use. |
| 4.0 | 2022 | Removed the mostly-obsolete `twine register`; dropped older Python versions. |
| 5.0 | 2024 | Maintenance release; further dropped end-of-life Python versions. |
| 6.0 | 2024 | Metadata parsing moved from `pkginfo` to `packaging`, reducing metadata-version breakage. |

## References

[^1]: Twine documentation — features, installation, and usage. https://twine.readthedocs.io/
[^2]: Python Packaging User Guide, "Uploading your project to PyPI" — rationale for twine over `setup.py upload`. https://packaging.python.org/tutorials/packaging-projects/
[^3]: Twine docs, "Configuration" — credential resolution order and `~/.pypirc`. https://twine.readthedocs.io/en/stable/
[^4]: PyPI, "Trusted Publishers" and 2FA requirements. https://docs.pypi.org/trusted-publishers/

## Tags

python, packaging, pypi, publishing, cli, distribution, wheel, sdist, devtools, pypa
