# spulec/freezegun

> Freezes Python's clock inside a test by monkeypatching the `datetime` and `time` modules.

[GitHub repo](https://github.com/spulec/freezegun) ·
[PyPI](https://pypi.org/project/freezegun/) ·
[License: Apache-2.0](https://github.com/spulec/freezegun/blob/master/LICENSE)

## Overview

FreezeGun is a test-support library that makes time-dependent Python code deterministic. Inside a `freeze_time` block, `datetime.datetime.now()`, `datetime.date.today()`, `time.time()`, `time.monotonic()`, `time.perf_counter()`, and the related `localtime`/`gmtime`/`strftime` calls all return a fixed instant instead of the wall clock[^1]. It has been the default answer to "how do I test code that reads the current time" in the Python ecosystem since 2012[^2], and remains widely depended on despite a newer, faster competitor (see Alternatives).

The library is aimed squarely at test suites, not production. Its central design decision — and the source of both its convenience and its problems — is that it patches the standard-library time functions *globally* by walking every already-imported module and swapping references to the real `datetime`/`date` classes for fake subclasses. This makes freezing transparent to code you did not write, but it also makes freezing a whole-interpreter side effect rather than a scoped one, which shows up as slowness in large codebases and as breakage in libraries that touch the clock in ways FreezeGun cannot intercept.

FreezeGun is stable and maintained rather than actively evolving: at ~4.5k stars with a broad reverse-dependency footprint, changes are conservative and backward-compatibility is prioritized over new capability. Treat it as infrastructure, not a project you follow for features.

## Getting Started

```bash
pip install freezegun
```

```python
from freezegun import freeze_time
import datetime

@freeze_time("2012-01-14")
def test_frozen():
    assert datetime.datetime.now() == datetime.datetime(2012, 1, 14)

# As a context manager, with a controllable clock:
def test_ticking():
    with freeze_time("2012-01-14 12:00:00") as frozen:
        assert datetime.datetime.now() == datetime.datetime(2012, 1, 14, 12, 0, 0)
        frozen.tick(delta=datetime.timedelta(hours=1))
        assert datetime.datetime.now() == datetime.datetime(2012, 1, 14, 13, 0, 0)
        frozen.move_to("2015-06-01")   # jump anywhere
```

Input parsing goes through `dateutil`, so human strings like `"Jan 14th, 2012"` work. `freeze_time` can also take a `datetime`, a `date`, a zero-arg callable, or a generator that yields successive instants[^1].

## Architecture / How It Works

The core is `freezegun/api.py`. On entry, `freeze_time`:

1. Builds `FakeDatetime` and `FakeDate` subclasses of the real `datetime.datetime` / `datetime.date`, whose `now()`/`today()`/`utcnow()` return the frozen instant. A custom metaclass provides `__instancecheck__`/`__subclasscheck__` so `isinstance(x, datetime.datetime)` still holds for both real and fake objects.
2. Replaces `datetime.datetime` and `datetime.date` themselves, plus `time.time`, `time.monotonic`, `time.perf_counter`, `time.localtime`, `time.gmtime`, `time.strftime`, and friends, with fakes.
3. Iterates over `sys.modules` and, for every loaded module holding a direct reference to the real `datetime`/`date` (e.g. a module that did `from datetime import datetime`), rebinds that attribute to the fake. On `stop()`, all of this is reversed.

Step 3 is why freezing is transparent: code that imported `datetime` before the freeze still sees frozen time. It is also why FreezeGun has an `ignore` list — some modules (`threading`, `Queue`, `selenium`, parts of `_pytest`, several others by default) must *not* have their clock swapped or they misbehave, and you can extend the list via `freezegun.configure(extend_ignore_list=[...])`[^1].

Monotonic clocks (`time.monotonic`, `time.perf_counter`) are frozen too, which is correct for reproducibility but dangerous for anything that schedules on elapsed monotonic time — hence the `real_asyncio=True` option, which lets an asyncio event loop keep seeing real monotonic time so `asyncio.sleep()` and timeouts do not hang[^1]. Clock behavior within a freeze is selectable: fully stopped (default), auto-advancing per call (`auto_tick_seconds`), advancing at real speed from the start point (`tick=True`), or manually stepped (`.tick()` / `.move_to()`).

## Production Notes

FreezeGun runs in test code, so "production" here means CI reliability and suite performance.

- **Per-freeze cost scales with imported modules.** Because start/stop walks `sys.modules` and rebinds references, the overhead of entering a frozen block grows with how much of your application is imported. In large suites that wrap thousands of tests in `freeze_time`, this is a measurable contributor to slow test runs — the most common reason teams migrate to `time-machine`, which patches at the C level and does not scan modules.
- **It cannot patch what it cannot reach.** C extensions that read the clock via the OS directly (rather than through Python's `time`/`datetime` objects), other processes, and the database server's own `NOW()`/`CURRENT_TIMESTAMP` are all unaffected. Freeze the Python side and your ORM-generated SQL timestamp still comes from Postgres's clock.
- **Third-party datetime types are not covered.** `pandas.Timestamp`, `numpy.datetime64`, and similar carry their own notion of "now"; FreezeGun does not freeze them.
- **`utcnow()` is a moving target.** `datetime.datetime.utcnow()` is deprecated from Python 3.12 onward; test code relying on it (and on FreezeGun faithfully faking it) should migrate to timezone-aware `datetime.now(tz=...)`.
- **Global state means concurrency hazards.** A freeze mutates interpreter-wide globals. Parallel test execution that shares an interpreter (threads, some `pytest-xdist` load modes are process-isolated and safe; in-process threading is not) can see one test's frozen clock leak into another.
- **Object identity gotcha.** Inside a freeze, `type(datetime.datetime.now())` is `FakeDatetime`, not `datetime.datetime`. `isinstance` is handled, but code doing exact `type(...) is datetime.datetime` comparisons, pickling, or C-level type checks can trip.
- **Default mutable arguments are frozen at import.** A function signature like `def f(default=date.today())` evaluates `today()` when the module is imported, long before your freeze — FreezeGun cannot retroactively change it.

## When to Use / When Not

**Use when:**
- You need to make existing time-reading code deterministic in tests without threading a clock dependency through your call sites.
- You want human-friendly freeze inputs, `move_to`/`tick` control, and drop-in decorator/context-manager/class ergonomics.
- Your suite's freeze overhead is not a bottleneck, or you value FreezeGun's maturity and huge install base over raw speed.

**Avoid (or prefer an alternative) when:**
- Test-suite performance matters and you freeze time heavily — `time-machine` is typically several times faster and avoids the module scan.
- The clock you care about lives outside Python (database, C library, subprocess) — inject a clock or use OS-level faking (`libfaketime`).
- You are writing new code and can afford it: passing a `now` callable / clock object into functions is more explicit and faster than any monkeypatch, and needs no test library at all.

## Alternatives

- adriangb/time-machine — C-based clock faking by Adam Johnson; same ergonomics, no `sys.modules` scan, substantially faster. Use it when freeze overhead shows up in your test timings.
- wolfcw/libfaketime — `LD_PRELOAD` shim that fakes time at the syscall level for the whole process, language-agnostic. Use it when non-Python or C-extension code reads the clock.
- ktosiek/pytest-freezegun — pytest fixture/marker wrapper around FreezeGun. Use it when you want `@pytest.mark.freeze_time` instead of decorators.
- python `unittest.mock.patch` on `datetime.now` directly — no dependency, but you must patch each import site by hand. Use it for a single narrow test where a full freeze is overkill.
- Dependency-injected clock (pass a `now` callable) — not a library; the design that removes the need for time mocking entirely in code you control.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1.x | 2012-12 | Initial release; global `datetime`/`time` monkeypatching[^2]. |
| 1.0.0 | 2020-11 | First stable major; API and typing stabilized. |
| 1.2–1.4 | 2022–2023 | `real_asyncio` for asyncio-safe monotonic time; `auto_tick_seconds`; ongoing Python-version support. |
| 1.5.x | 2024–2025 | Current line; newer-CPython support, older-Python drops. Latest push 2025-08-19[^3]. |

Exact intermediate release dates vary — see the PyPI release history for the authoritative timeline[^4].

## References

[^1]: FreezeGun README — usage, options (`tick`, `auto_tick_seconds`, `move_to`, `real_asyncio`), and default `ignore` list. https://github.com/spulec/freezegun/blob/master/README.rst
[^2]: Repository created 2012-12-11 (GitHub API `created_at`). https://github.com/spulec/freezegun
[^3]: GitHub API repo metadata (stars, license Apache-2.0, last push 2025-08-19), retrieved 2026-07. https://api.github.com/repos/spulec/freezegun
[^4]: FreezeGun release history on PyPI. https://pypi.org/project/freezegun/#history

## Tags

python, testing, mocking, datetime, time, test-utilities, monkeypatching, pytest, unittest, freeze-time
