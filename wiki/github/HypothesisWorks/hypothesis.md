# HypothesisWorks/hypothesis

> Property-based testing for Python: describe the space of valid inputs, and let the library hunt for the counterexample you didn't think of — then shrink it to the minimal reproduction.

[GitHub repo](https://github.com/HypothesisWorks/hypothesis) ·
[Official website](https://hypothesis.works) ·
[License: MPL-2.0](https://github.com/HypothesisWorks/hypothesis/blob/master/LICENSE.txt)

## Overview

Hypothesis is the dominant property-based testing (PBT) library for Python, created by David R. MacIver and first released as 1.0 in 2015[^1]. Instead of writing example-based tests (`assert f(2) == 4`), you write a property that should hold for a whole class of inputs, describe that class with a *strategy*, and Hypothesis generates hundreds of examples — including boundary values, empty collections, NaNs, and Unicode edge cases — trying to falsify the property. It is descended in spirit from Haskell's QuickCheck but diverges sharply in its generation and shrinking engine[^2].

Its defining feature is not generation but *shrinking*: when a test fails, Hypothesis does not report the random 400-element list that broke it — it reduces the failure to the simplest input that still fails (often a one- or two-element case) and reports that. This turns a test failure into a near-minimal bug reproduction. The second defining feature is the *example database*: failing inputs are persisted to `.hypothesis/` on disk and replayed first on the next run, so a flaky bug found once is deterministically re-found until fixed[^3].

The core tension is cost versus coverage. A property test is not free: each `@given` test runs many times (100 examples by default), generation and shrinking add wall-clock time, and the randomized nature means a test that passes today can fail tomorrow when a new example hits an untested path. That last property is the point — but it collides with teams that expect CI to be perfectly reproducible run-to-run.

## Getting Started

```bash
pip install hypothesis
# optional integrations: pip install "hypothesis[numpy,pandas,django]"
```

```python
from hypothesis import given, strategies as st


# Property: encoding then decoding round-trips to the original string.
@given(st.text())
def test_decode_inverts_encode(s):
    assert s.encode("utf-8").decode("utf-8") == s


# Property: my_sort must match the builtin for every list of integers.
@given(st.lists(st.integers()))
def test_matches_builtin(xs):
    assert my_sort(xs) == sorted(xs)
```

If `my_sort` is buggy (say it dedupes), Hypothesis reports the minimal falsifying example, e.g. `xs=[0, 0]`, not whatever large random list first triggered it.

## Architecture / How It Works

A strategy (`st.integers()`, `st.lists(...)`, `st.builds(MyClass, ...)`) is a recipe for drawing values. Under the hood it does *not* generate a Python value directly. The engine — historically named **Conjecture** — generates an underlying *choice sequence* (originally a byte stream, later a typed sequence of choices), and each strategy is a decoder that maps that sequence into a Python object. Shrinking operates on the choice sequence, not the decoded value: make the underlying sequence lexicographically smaller/shorter, re-decode, re-run the test. Because every strategy shares this substrate, shrinking is generic and does not need per-type shrink logic the way classic QuickCheck did[^2].

Key surfaces built on the core:

- **`@given`** — turns a test function into a driver that runs the body across many generated inputs within a single pytest test invocation.
- **`@settings`** — controls `max_examples`, `deadline`, `phases`, `derandomize`, and health checks per test or via named profiles.
- **`assume(cond)`** — rejects a generated input mid-test; too many rejections triggers the `filter_too_much` health check.
- **`@example(...)`** — pins explicit inputs that always run, useful for regressions and documentation.
- **Stateful testing** — `RuleBasedStateMachine` generates *sequences of method calls* and shrinks the sequence, for testing stateful systems (caches, data structures, state machines).
- **`target()`** — targeted PBT that steers generation toward maximizing a metric, a lightweight coverage-guidance mechanism.
- **Ghostwriter** — `hypothesis write module.func` inspects a function's signature/types and emits a starter test file[^4].

Integrations live under `hypothesis.extra` (numpy arrays, pandas frames, Django models, `dateutil`, `zoneinfo`, the Array API). The project ships with an unusual release process: essentially every merged PR that touches behavior triggers an automated PyPI release, so 6.x has accumulated thousands of patch versions[^5].

## Production Notes

- **The `deadline` footgun.** By default each example must complete within 200 ms or Hypothesis raises `DeadlineExceeded`. On slow CI runners, first-call JIT warmup, or tests doing real I/O, this surfaces as intermittent failures unrelated to the property. The fix is `@settings(deadline=None)` (or a larger value) for tests that legitimately vary in timing.
- **Function-scoped pytest fixtures do not reset per example.** Because all examples run inside one test-function call, a function-scoped fixture is set up once and shared across every generated input. State leaks between examples and Hypothesis emits a health-check error. Use fresh objects built inside the test, or design fixtures to be immutable.
- **The example database is local and unshared by default.** `.hypothesis/examples` lives in the working directory. A failure found on a developer's machine is replayed there but is *not* automatically available in CI (or vice versa) unless you configure a shared database backend or commit/transport the entries. Teams surprised by "it fails locally but CI is green" are usually hitting this.
- **Reproducibility.** Randomized generation means the specific examples differ run-to-run. For a bug you must reproduce exactly, Hypothesis prints a `@reproduce_failure` blob, or you can set `derandomize=True` / a fixed `HYPOTHESIS_SEED`. Do not assume a green run proves absence of the bug the next run will find.
- **Health checks and shrinking cost.** Large/expensive strategies can trip `data_too_large`, `too_slow`, or `filter_too_much`. Pathological shrinking on a slow test body can make a single failure take minutes to minimize; cap `max_examples` and keep test bodies fast.
- **`Flaky` errors.** If a test passes during generation but fails on replay (or vice versa), Hypothesis raises `Flaky`, which almost always indicates hidden global state, ordering dependence, or nondeterminism in the code under test — a real signal, not a library bug.

## When to Use / When Not

**Use when:**
- The code has invariants expressible over a class of inputs: round-trips (encode/decode, serialize/parse), idempotence, commutativity, agreement with a reference implementation, or "never crashes / never violates a postcondition."
- You're testing parsers, serializers, numeric code, data structures, or stateful systems where hand-picked examples miss edge cases.
- You want failing cases to arrive pre-minimized and auto-persisted for regression.

**Avoid when:**
- The expected output for each input can only be stated by re-implementing the function — you often end up with a tautological test. Prefer example tests or a genuinely independent oracle.
- CI must be bit-for-bit reproducible and a nondeterministic new failure is organizationally unacceptable (mitigable with `derandomize`, but friction remains).
- The unit is trivial or has a small enumerable input domain — `pytest.mark.parametrize` is simpler and clearer.

## Alternatives

- google/atheris — coverage-guided (libFuzzer) fuzzer for Python; use when the goal is finding crashes/security bugs by mutation rather than asserting declared properties.
- schemathesis/schemathesis — API/OpenAPI test generator built on top of Hypothesis; use when the subject under test is a web service, not a Python function.
- pschanely/CrossHair — symbolic-execution contract checker; use when you want exhaustive path analysis over `assert`/contracts instead of randomized sampling.
- dubzzz/fast-check — property-based testing for JavaScript/TypeScript; use when the code under test is JS/TS.
- proptest-rs/proptest — property-based testing for Rust with integrated shrinking; the closest same-shape tool in that ecosystem.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2015-03 | First stable release; `@given` + strategies core[^1]. |
| 3.0 | 2016-03 | Conjecture engine — shrinking moved onto an internal byte stream[^2]. |
| 4.0 | 2019-02 | API cleanup; deprecations removed. |
| 5.0 | 2020-01 | Dropped Python 2 support[^6]. |
| 6.0 | 2021-01 | Current major line; ongoing continuous per-PR releases[^5]. |

## References

[^1]: David R. MacIver, project history and origins. https://hypothesis.works/articles/how-hypothesis-works/
[^2]: "How Hypothesis Works" — Conjecture engine and choice-sequence shrinking. https://hypothesis.works/articles/how-hypothesis-works/
[^3]: Hypothesis documentation — the example database. https://hypothesis.readthedocs.io/en/latest/database.html
[^4]: Hypothesis documentation — Ghostwriter. https://hypothesis.readthedocs.io/en/latest/ghostwriter.html
[^5]: Hypothesis contributing / continuous-release process (RELEASE.rst per PR). https://github.com/HypothesisWorks/hypothesis/blob/master/CONTRIBUTING.rst
[^6]: Hypothesis changelog (version history). https://hypothesis.readthedocs.io/en/latest/changes.html

## Tags

python, property-based-testing, testing, fuzzing, pytest, quickcheck, test-generation, shrinking, quality-assurance
