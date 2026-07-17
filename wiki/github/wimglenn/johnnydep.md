# wimglenn/johnnydep

> A CLI and library that prints the dependency tree of a Python distribution by reading each package's published metadata from an index.

[GitHub repo](https://github.com/wimglenn/johnnydep) ·
[License: MIT](https://github.com/wimglenn/johnnydep/blob/main/LICENSE)

## Overview

johnnydep answers one narrow question: "if I were to install package X, what
does it declare that it needs?" It is a single-author project by Wim Glenn,
first published in 2018[^1]. You give it a requirement (`requests`,
`boto3==1.34`, `flask[async]`), and it walks the declared dependency graph,
printing a tree with each node's version specifier and one-line summary.

The important distinction — and the source of most confusion — is that
johnnydep is an *inspection* tool, not a resolver or installer. By default it
shows the `Requires-Dist` specifiers that each distribution *declares*
(`urllib3<3,>=1.26`), not a single resolved, installable set of pins. It only
attempts resolution when you ask for it explicitly (`--output-format pinned`),
and even then it does not run pip's full backtracking solver — it picks the
newest version satisfying each specifier as it descends[^2]. Treat its tree as
"what the metadata says," not "what pip will install."

Its defining tradeoff is that it works entirely from a package index (PyPI by
default) *before* installation, unlike tools that inspect an already-installed
environment. That makes it useful for auditing a package you have not committed
to yet — but it pays for that reach by downloading real distributions to read
their metadata, which makes it network-heavy and slow relative to
environment-scanning alternatives.

## Getting Started

```bash
pip install johnnydep
# no install needed with uv:
uvx johnnydep requests
```

```bash
# Human-readable tree (default)
johnnydep requests

# Best-effort resolved pins, one per line
johnnydep ipython --output-format pinned

# Machine-readable output for tooling
johnnydep boto3 --output-format json --fields name version_latest requires
```

As a library, the unit of work is the `JohnnyDist` object, which lazily fetches
and caches metadata as you traverse it:

```python
from johnnydep import JohnnyDist

dist = JohnnyDist("requests")
print(dist.version_latest)     # newest version on the index
for req in dist.requires:      # declared Requires-Dist specifiers
    print(req)                 # e.g. "urllib3<3,>=1.26"
print(dist.pinned)             # e.g. "requests==2.32.3"
```

## Architecture / How It Works

johnnydep does not parse a lockfile or scan `site-packages`. For each
requirement, it asks an index for the distribution and reads that
distribution's core metadata to extract `Requires-Dist`, then recurses into
each child requirement[^3].

- **Metadata acquisition.** It uses pip's download machinery to fetch the
  distribution for a requirement without installing it, preferring a wheel when
  one is available because a wheel's `METADATA` is static and cheap to read.
- **Source distributions are the expensive path.** If only an sdist is
  available and it lacks statically declared metadata, obtaining
  `Requires-Dist` may require a PEP 517 metadata-preparation build, which can
  execute the package's own build backend. This is slower and carries the usual
  "arbitrary code from an sdist" caveat.
- **Caching.** Downloaded artifacts and computed metadata are cached on disk so
  repeated queries in a session (and across runs) avoid re-fetching. The cache
  is keyed on the resolved distribution, not on wall-clock freshness.
- **Environment markers.** Dependencies gated by markers
  (`; python_version < "3.11"`, `; sys_platform == "win32"`) are evaluated
  against a target environment. By default that is the current interpreter, so
  the same query can produce different trees on different machines; a target
  environment can be supplied explicitly.
- **Extras.** Requesting `flask[async]` expands the extra's conditional
  requirements into the tree.
- **Output layer.** The traversed graph is rendered through pluggable formats —
  the box-drawing tree, `pinned`, `json`, `yaml`, and a flat field table — via
  small serialisation helpers over the same `JohnnyDist` nodes.

The whole design leans on the standards stack: PEP 427 wheels, core-metadata
`Requires-Dist`, and PEP 508 requirement/marker parsing (via `packaging`).
johnnydep is essentially a thin traversal-and-render layer over "download the
dist, read its metadata."

## Production Notes

- **It is network- and disk-heavy by design.** Because it downloads real
  distributions to read metadata, a deep tree (`boto3`, `apache-airflow`) can
  pull down many megabytes and take noticeably longer than an
  environment-scanning tool. This makes it a poor fit for tight CI loops unless
  the cache is warmed and persisted.
- **The default tree is not an install plan.** Nodes show *declared*
  specifiers, and the same transitive package can appear under multiple parents
  with different ranges. Do not read the default output as "these exact
  versions will be installed." Use `--output-format pinned` when you need a
  single resolved set, and remember its resolution is newest-satisfying, not
  pip's backtracking solver — the two can disagree on hard conflicts.
- **sdist builds can run code.** For packages that ship no wheel and no static
  metadata, metadata extraction may invoke the build backend. Be aware of this
  when pointing johnnydep at untrusted or unusual requirements.
- **Results are environment-dependent.** Marker evaluation against the current
  interpreter means Python version and platform change the tree. Pin the target
  environment if you need reproducible output across machines.
- **Cache staleness.** Cached metadata speeds things up but is keyed on
  distribution identity; if you expect a just-published version and see an old
  one, the cache or index mirror is the usual culprit.
- **Scope creep is not the goal.** It intentionally does not install, lock, or
  manage environments. Reaching for it as a substitute for a resolver leads to
  surprises on version conflicts.

## When to Use / When Not

**Use when:**
- You want to inspect a package's declared dependencies *before* installing it.
- You're auditing what a candidate dependency drags in (supply-chain review,
  license sweeps, "why is this so big").
- You need machine-readable dependency metadata (`json`/`yaml`/`pinned`) for a
  script or ad-hoc tooling, driven off an index rather than a live environment.

**Avoid when:**
- You want the dependency tree of what is *already installed* — that is a
  different question, answered faster by an environment scanner.
- You need an authoritative, conflict-resolved lockfile — use a real resolver.
- You're in a latency-sensitive CI step and cannot afford network downloads.

## Alternatives

- tox-dev/pipdeptree — renders the dependency tree of packages already
  installed in the current environment; use it when you want to see what you
  actually have, not what an index declares.
- astral-sh/uv — fast resolver/installer with `uv tree`; use it when you want a
  real resolved graph and lockfile, not a pre-install inspection.
- jazzband/pip-tools — `pip-compile` produces pinned lockfiles from
  requirements; use it when the goal is reproducible installs rather than
  browsing declared metadata.
- python-poetry/poetry — full project dependency management with a solver and
  lockfile; use it when you own the project and want end-to-end management.
- pypa/pipgrip — a CLI dependency resolver/tree that, like johnnydep, works
  from the index; use it when you specifically want backtracking resolution
  output.

## History

| Milestone | Date | Notes |
|-----------|------|-------|
| First published | 2018-02 | Repo created; released on PyPI as `johnnydep`[^1]. |
| Ongoing 1.x line | — | Single-maintainer project, iterative releases tracking the packaging ecosystem (wheels, `packaging`, pip's resolver). |
| Actively maintained | 2026-07 | Most recent push 2026-07-02; ~550 stars, MIT-licensed. |

## References

[^1]: johnnydep on PyPI. https://pypi.org/project/johnnydep/
[^2]: README, note on pip's resolver history and the meaning of the `pinned` output. https://github.com/wimglenn/johnnydep#readme
[^3]: Python Packaging core metadata specification (`Requires-Dist`). https://packaging.python.org/specifications/core-metadata/

## Tags

python, cli, dependency-tree, packaging, pypi, dependency-analysis, developer-tools, metadata, supply-chain, wheels
