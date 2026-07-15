# coveragepy/coveragepy

> The de facto code coverage tool for Python — line and branch measurement built on the interpreter's own tracing hooks.

[GitHub repo](https://github.com/coveragepy/coveragepy) ·
[Official docs](https://coverage.readthedocs.io) ·
[License: Apache-2.0](https://github.com/coveragepy/coveragepy/blob/main/LICENSE.txt)

## Overview

Coverage.py measures which lines (and optionally which branches) of your Python
code actually execute, typically while a test suite runs. It is maintained by
Ned Batchelder, who took over the project around 2004 after Gareth Rees's
original 2001 implementation[^1]. It is the measurement engine underneath nearly
every Python coverage workflow: `pytest-cov`, Codecov, Coveralls, and most CI
setups are thin wrappers or consumers of its output rather than independent
tools.

The project is old, singularly maintained, and unusually stable in scope: it
counts executed lines and reports them. That focus is the point. The defining
tension is cost versus fidelity — accurate coverage means observing every line
the interpreter runs, which historically imposed a real runtime tax (the trace
callback fires per line). Recent Python versions expose `sys.monitoring`
(PEP 669), and coverage.py can now use it to cut that overhead substantially,
but the classic C tracer remains the default on older interpreters[^2].

As of 2026 the GitHub repo has ~3,400 stars and ~500 forks with commits landing
within the last day — a small star count for how load-bearing it is, since most
developers use it through a wrapper and never star it directly. The code moved
from a Mercurial/Bitbucket history to GitHub around 2018, which is why the
repository's creation date long postdates the project.

## Getting Started

```bash
pip install coverage
```

```bash
# Run your suite under coverage, then report
coverage run -m pytest
coverage report -m           # terminal report with missing line numbers
coverage html                # browsable htmlcov/ report

# Or measure a plain script
coverage run --branch my_script.py
```

Configuration lives in `pyproject.toml` (or `.coveragerc` / `setup.cfg` /
`tox.ini`):

```toml
[tool.coverage.run]
branch = true
source = ["mypackage"]

[tool.coverage.report]
fail_under = 90
show_missing = true
exclude_lines = ["pragma: no cover", "if TYPE_CHECKING:"]
```

Most pytest users install `pytest-cov` and pass `--cov=mypackage` instead of
invoking `coverage run` directly; it drives the same engine.

## Architecture / How It Works

Coverage.py has three phases: **collection**, **storage**, and **reporting**.

**Collection.** During execution a tracer records which lines run. Two exist: a
C extension (`CTracer`) used by default for speed, and a pure-Python `PyTracer`
fallback. Both historically hook `sys.settrace`, the standard-library trace
function that fires a callback on each executed line. Since coverage 7.4 an
alternative core built on `sys.monitoring` (PEP 669, Python 3.12+) can be
selected via `COVERAGE_CORE=sysmon`, which lets the interpreter disable
instrumentation on already-seen lines and dramatically lowers overhead[^2].

**Static analysis.** To know which lines *could* run, coverage.py parses each
file's AST and bytecode to distinguish executable lines from comments, blanks,
and docstrings, and — for branch coverage — enumerates the possible arc
transitions between lines. Reported coverage is executed executable lines ÷ all
executable lines; branch coverage additionally checks that both sides of each
conditional arc were taken.

**Storage.** Since coverage 5.0 the `.coverage` data file is a SQLite database,
replacing the older pickle/JSON format[^3]. This enabled dynamic *contexts*
(labeling which test executed which line) and made parallel data merging
robust, at the cost of a hard format break with pre-5.0 files.

**Reporting.** From collected data coverage.py renders terminal, HTML, XML
(Cobertura), JSON, and LCOV outputs. XML/JSON/LCOV are what CI services and
tools like `diff-cover` consume. The reporting layer is decoupled from
collection, so you can run tests on one machine and report elsewhere.

A plugin API allows measuring non-Python execution (e.g. Django or Mako
template coverage) by supplying custom file reporters and tracers.

## Production Notes

**Subprocess measurement is the classic footgun.** Code launched in a
subprocess is *not* measured unless you set `COVERAGE_PROCESS_START` and invoke
`coverage.process_startup()` via a `.pth` file. Teams routinely ship "100%
coverage" that silently excludes everything running in worker processes.

**Parallel mode requires an explicit combine step.** With `parallel = true`
each process writes a uniquely-suffixed data file (`.coverage.<host>.<pid>.<n>`);
you must run `coverage combine` before reporting or you get partial or empty
results. This bites `pytest-xdist` and multiprocessing suites constantly.

**Concurrency setting must match your runtime.** If you use gevent, eventlet,
greenlet, or `multiprocessing`, the `concurrency` config must name it, or lines
are misattributed across green threads and coverage under-reports.

**Coverage must be the outermost layer.** Anything that executes at import time
before `coverage run` starts — module-level code, some plugin registration —
is invisible. This makes measuring a package's own import side effects awkward.

**Overhead.** The default C tracer typically adds a meaningful slowdown
(commonly ~2–3× on trace-heavy code, more with `--branch`). On Python 3.12+,
`COVERAGE_CORE=sysmon` can bring this close to zero for line coverage and is
the recommended setting for large suites, though branch support under sysmon
matured later than line support[^2].

**SQLite on network filesystems.** The `.coverage` file inherits SQLite's
locking behavior; writing it to NFS or a container-shared mount can produce
`database is locked` errors under contention. Keep it on local disk and combine
afterward.

**Upgrade breaks.** 5.0 changed the data format (old files unreadable); 6.0
dropped Python 2 and 3.5; 7.0 continued trimming old interpreters. Pin the major
version in CI if you aggregate `.coverage` files across jobs.

## When to Use / When Not

**Use when:**
- You write Python and want line or branch coverage, in essentially any test runner.
- You need coverage output in a standard format (XML/JSON/LCOV) for Codecov, Coveralls, or diff-based gating.
- You want per-test contexts to see *which* test covered a line.

**Avoid / look elsewhere when:**
- Tracing overhead is unacceptable and you're on an older Python without `sys.monitoring` — a sampling or bytecode-instrumentation tool may fit better.
- You need coverage for a compiled language or Python C extensions — coverage.py measures Python source only (use `gcov`/`llvm-cov` for the C side).
- You only care about changed lines in a PR — pair it with a diff tool rather than reading whole-project percentages.

## Alternatives

- plasma-umass/slipcover — near-zero-overhead line coverage via bytecode instrumentation; use when coverage.py's tracing cost is prohibitive and you can live with fewer features.
- pytest-dev/pytest-cov — not a replacement but the standard pytest integration; use when you're on pytest and want coverage wired into fixtures and xdist instead of calling `coverage run` yourself.
- Bachmann1234/diff-cover — use when you only want to enforce coverage on lines changed in a PR; it consumes coverage.py's XML/JSON output.
- codecov/codecov-action — use when you want hosted dashboards and PR annotations across many repos; it aggregates coverage.py reports rather than measuring anything itself.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2001 | Original implementation by Gareth Rees[^1]. |
| — | ~2004 | Maintenance taken over by Ned Batchelder[^1]. |
| 4.0 | 2015-09 | Major internals overhaul; improved branch coverage. |
| 5.0 | 2019-12 | SQLite data file, dynamic contexts[^3]. |
| 6.0 | 2021-10 | Dropped Python 2 and 3.5. |
| 7.0 | 2022-12 | Current major line; continued dropping legacy Pythons. |
| 7.4 | 2024-01 | `sys.monitoring` core (`COVERAGE_CORE=sysmon`) added[^2]. |

## References

[^1]: Ned Batchelder, "coverage.py" project overview and history. https://nedbatchelder.com/code/coverage/
[^2]: Coverage.py docs, "Configuration / COVERAGE_CORE" and PEP 669 sys.monitoring support. https://coverage.readthedocs.io/en/latest/config.html
[^3]: Coverage.py changelog, 5.0 release (SQLite data file, contexts). https://coverage.readthedocs.io/en/latest/changes.html

## Tags

python, code-coverage, testing, test-tooling, pytest, sys-monitoring, branch-coverage, ci, quality, cli
