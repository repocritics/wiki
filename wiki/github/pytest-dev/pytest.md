# pytest-dev/pytest

> The de facto testing framework for Python — plain `assert` statements, fixtures via dependency injection, and a plugin surface that most of the ecosystem builds on.

[GitHub repo](https://github.com/pytest-dev/pytest) ·
[Official website](https://pytest.org) ·
[License: MIT](https://github.com/pytest-dev/pytest/blob/main/LICENSE)

## Overview

pytest is a test framework for Python that descends from the `py` library's `py.test` runner (Holger Krekel, ~2004) and became a standalone project with pytest 2.0 in 2010[^1]. It has since displaced `unittest` as the default choice for most Python projects: at ~14k GitHub stars and 1300+ published external plugins[^2], its conventions (`test_*.py` discovery, fixtures, `@pytest.mark.parametrize`) are effectively the ecosystem's lingua franca, even for projects that ship their tests via `unittest.TestCase`.

The defining design bet is **plain `assert`**. Where `unittest` requires `self.assertEqual(a, b)`, pytest rewrites the AST of test modules at import time so a bare `assert inc(3) == 5` produces a full introspected failure message showing both operands. This one decision removes most of the ceremony from writing tests, and it is why pytest reads like ordinary Python rather than a testing DSL.

The tension is that pytest is powerful because it is deeply magical. Fixtures are resolved by name-matching function arguments, `conftest.py` files are auto-loaded by directory, plugins hook into collection and execution through implicit registration, and assertion rewriting is an import-time bytecode transform. When it works it is invisible; when it breaks — import-mode collisions, fixture-scope surprises, a plugin that silently changes behavior — the failure is often several layers below the test you wrote.

## Getting Started

```bash
pip install pytest
```

```python
# content of test_sample.py
def inc(x):
    return x + 1

def test_answer():
    assert inc(3) == 5      # plain assert — pytest introspects the operands
```

```bash
$ pytest -q
```

Fixtures and parametrization, the two features you reach for immediately:

```python
import pytest

@pytest.fixture
def db():
    conn = connect(":memory:")
    yield conn                 # teardown runs after the test
    conn.close()

@pytest.mark.parametrize("value,expected", [(3, 4), (10, 11), (-1, 0)])
def test_inc(db, value, expected):
    assert inc(value) == expected
```

## Architecture / How It Works

pytest is four subsystems stitched together by a hook framework:

1. **Collection.** pytest walks the filesystem from a computed `rootdir`, imports modules matching `python_files`, and builds a tree of `Node` objects — `Session` → `Module` → `Class` → `Function`. Discovery rules are convention-based: files named `test_*.py`, functions `test_*`, classes `Test*` **without an `__init__`**. Missing any of these silently drops the tests from the run.

2. **Fixtures.** A fixture is a function decorated with `@pytest.fixture`; a test requests it by naming it as a parameter. pytest resolves the dependency graph, honors `scope=` (`function`/`class`/`module`/`package`/`session`), and runs teardown via `yield` finalizers in reverse order. Fixtures can request other fixtures, and `conftest.py` files make them available to a whole directory subtree without import.

3. **Assertion rewriting.** At import, pytest installs a [PEP 302] import hook that rewrites `assert` statements in test modules (and registered plugins) into code that captures sub-expression values. This only applies to collected test modules and plugins registered via `register_assert_rewrite` — helper modules imported normally get plain `AssertionError` with no introspection.

4. **The plugin system (`pluggy`).** pytest's extensibility is a separate library, `pluggy`[^3], based on named hook specifications (`pytest_collection_modifyitems`, `pytest_runtest_setup`, etc.). Plugins — including built-ins like `fixtures`, `mark`, `capture`, and third-party ones — register hook implementations that pluggy calls in a defined order. `conftest.py` is itself a local, auto-discovered plugin.

The coupling story: fixtures, marks, parametrization, and assertion rewriting all flow through the same collection/execution pipeline, and third-party plugins hook the same pipeline. That is why a single incompatible plugin can change collection order, mangle output, or break teardown across an entire suite.

## Production Notes

**Import modes are the top footgun.** The default `prepend` import mode inserts each test file's directory onto `sys.path`, which means two `test_foo.py` files in different directories without `__init__.py` collide as the same module name and raise a confusing import error. The fixes are: add `__init__.py` packages, or switch to `--import-mode=importlib`[^4]. New projects should strongly consider `importlib` mode from the start; it avoids `sys.path` mutation entirely but requires unique test module paths.

**Fixture scope and finalization.** Session/module-scoped fixtures that hold real resources (DB connections, temp servers) are shared across tests — mutating shared state in one test leaks into the next. Teardown ordering with nested `yield` fixtures is reverse-of-setup; getting this wrong produces resource leaks that only surface under `-p no:randomly` reordering or `pytest-xdist`.

**Parallelism is not built in.** `pytest -n auto` requires the third-party `pytest-xdist` plugin. Under xdist, tests run in separate worker processes, so any test relying on shared in-process state, ordering, or a session fixture's side effects will fail non-deterministically. Test isolation debt is invisible until you parallelize.

**Assertion rewriting gaps.** Assertions in helper/utility modules are *not* rewritten unless you call `pytest.register_assert_rewrite("mypkg.helpers")` before import. Teams building custom assertion helpers routinely lose introspection and don't notice.

**Plugin churn on major upgrades.** pytest's major versions (5→6→7→8) removed deprecated hooks and internal APIs that plugins depended on. Upgrades frequently break on a lagging plugin (`pytest-django`, `pytest-asyncio`, `pytest-cov`) rather than on your own tests. Pin plugin versions and upgrade pytest and its plugins together, not independently.

**Collection cost.** Large suites (10k+ tests) can spend significant wall-clock time in collection and fixture setup before the first test runs. `--co` (collect-only) profiles it; expensive module-level imports and session fixtures are the usual culprits.

## When to Use / When Not

**Use when:**
- You want low-ceremony tests in idiomatic Python with rich failure output.
- You need parametrization, fixtures with dependency injection, and a mature plugin ecosystem (coverage, async, Django, snapshots, property-based via Hypothesis).
- You have existing `unittest.TestCase` suites — pytest runs them out of the box and lets you migrate incrementally.

**Avoid when:**
- You cannot add a dependency (locked-down environments, stdlib-only constraints) — `unittest` ships with Python.
- You need keyword-driven acceptance tests for non-developers — Robot Framework fits that audience better.
- Your team is allergic to implicit magic and prefers everything explicit — pytest's fixture/plugin resolution will feel opaque.

## Alternatives

- python/cpython (`unittest`) — use when you need zero dependencies or a stdlib-only test runner; verbose `self.assert*` API, no fixtures/parametrization without boilerplate.
- HypothesisWorks/hypothesis — property-based testing; complements rather than replaces pytest, integrates as a pytest plugin.
- robotframework/robotframework — use for keyword-driven acceptance/BDD tests owned by non-programmers.
- nose2 — the successor to the abandoned `nose`; use only for legacy suites, largely superseded by pytest.
- tox-dev/tox — environment/matrix orchestration around your test runner, not a framework itself; commonly paired with pytest.

## History

| Version | Date | Notes |
|---------|------|-------|
| py.test | 2004 | Shipped inside the `py` library (Holger Krekel)[^1]. |
| 2.0 | 2010-11 | Split into a standalone `pytest` distribution. |
| 3.0 | 2016-08 | `approx`, improved parametrization, warnings capture. |
| 5.0 | 2019-06 | Dropped Python 2 support; Python 3.5+ only[^5]. |
| 6.0 | 2020-07 | `pyproject.toml` config support, docs overhaul. |
| 7.0 | 2022-02 | New `--import-mode=importlib` maturity, `pytest.Stash`. |
| 8.0 | 2024-01 | Reworked collection, stricter deprecations, Python 3.8+[^6]. |

Current releases require Python 3.10+ or PyPy3. The project is actively maintained under the pytest-dev org, with a broad maintainer team rather than a single BDFL, funded partly through Open Collective and Tidelift.

## References

[^1]: pytest history — the project grew out of the `py` library's `py.test` tool; standalone `pytest` distribution from 2.0. https://docs.pytest.org/en/stable/history.html
[^2]: pytest README and plugin list — "over 1300+ external plugins." https://docs.pytest.org/en/latest/reference/plugin_list.html
[^3]: pluggy — pytest's underlying hook/plugin framework. https://pluggy.readthedocs.io/
[^4]: pytest docs, "Import modes" — `prepend` vs `append` vs `importlib`. https://docs.pytest.org/en/stable/explanation/pythonpath.html
[^5]: pytest 5.0 release notes — Python 2 dropped. https://docs.pytest.org/en/stable/changelog.html
[^6]: pytest 8.0 changelog. https://docs.pytest.org/en/stable/changelog.html

## Tags

python, testing, unit-testing, test-framework, fixtures, plugin-architecture, assertion-rewriting, pytest, developer-tools, quality-assurance
