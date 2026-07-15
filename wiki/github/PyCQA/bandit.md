# PyCQA/bandit

> An AST-based static analyzer that flags common security anti-patterns in Python source — fast, low-setup, and heuristic rather than sound.

[GitHub repo](https://github.com/PyCQA/bandit) ·
[Official docs](https://bandit.readthedocs.io) ·
[License: Apache-2.0](https://github.com/PyCQA/bandit/blob/main/LICENSE)

## Overview

Bandit scans Python code for a fixed catalog of known-insecure patterns:
`eval`, `pickle.load`, `subprocess(..., shell=True)`, hardcoded passwords,
weak hashes (MD5/SHA1), disabled TLS verification, SQL built by string
concatenation, and so on. It parses each file into a Python AST and runs a
set of plugin checks against the nodes, then emits a report with a severity
and confidence rating per finding. It began inside the OpenStack Security
Project and was later rehomed to PyCQA, the group that also maintains flake8,
pylint, and isort[^1].

The defining characteristic — and the source of most complaints — is that
Bandit is pattern-matching, not taint analysis. It has no dataflow or
inter-procedural reasoning: it cannot tell whether the string passed to a
SQL query is attacker-controlled or a constant. So a check like B608
(hardcoded SQL) fires on the *shape* of the code, not on a proven
vulnerability. This makes Bandit fast and dependency-light, but it also means
findings are candidates to review, not confirmed bugs. Teams that treat its
output as a security gate without triage tend to drown in false positives and
end up sprinkling `# nosec` everywhere.

Its value is as a cheap first-pass linter that catches careless mistakes and
enforces house rules (no `assert` in production paths, no `yaml.load` without
`SafeLoader`) across a codebase or in CI, not as a substitute for a real
SAST engine with taint tracking.

## Getting Started

```bash
pip install bandit
# optional extras:
pip install "bandit[toml]"    # read config from pyproject.toml
pip install "bandit[sarif]"   # SARIF output for code-scanning dashboards
```

```bash
# recursively scan a package, medium+ severity only, medium+ confidence
bandit -r myproject/ -ll -ii

# emit machine-readable output for CI
bandit -r myproject/ -f sarif -o bandit.sarif
```

```python
# example.py — Bandit flags line 3 as B602 (subprocess shell=True)
import subprocess

def run(user_cmd):
    subprocess.call(user_cmd, shell=True)   # >> Issue: [B602] ... Severity: High

    # suppress a reviewed finding inline:
    subprocess.call("ls -la", shell=True)   # nosec B602 - static command, reviewed
```

## Architecture / How It Works

Bandit's pipeline is deliberately simple. For each target file it (1) reads
the source, (2) compiles it with Python's built-in `ast` module, (3) walks the
tree, and (4) dispatches each node to the plugins registered for that node
type. Plugins are ordinary Python functions decorated with `@test_id` and a
`@checks(...)` node-type filter; they return an `Issue` (with severity,
confidence, CWE, and text) or nothing[^2].

Checks come in two flavors. **Test plugins** (the `B1xx`–`B7xx` families)
inspect call sites, imports, and expressions — e.g. B303 (weak MD5/SHA1),
B301 (pickle), B501 (requests without cert validation), B701 (jinja2
autoescape off). **Blacklist plugins** are a lookup table of banned imports
and function calls, so importing `telnetlib` or calling `eval` matches by name
without bespoke logic. Every finding carries an **ID** (e.g. `B602`), a
**severity** (LOW/MEDIUM/HIGH), and a **confidence** (LOW/MEDIUM/HIGH); the
`-l`/`-i` flags filter on those two axes independently.

Because analysis is per-file and AST-only, Bandit has no view across module
boundaries. It cannot follow a value from a web handler into a `subprocess`
call, so it errs toward flagging syntactically suspicious constructs and
leaving the human to judge reachability. Configuration lives in a `.bandit`
INI file, a YAML file, or `[tool.bandit]` in `pyproject.toml` (with the `toml`
extra); config controls which tests run (`tests`/`skips`), excluded paths, and
per-test options. The plugin registry is exposed through setuptools entry
points, so third parties can ship additional checks as installable packages.

## Production Notes

- **False positives are the operating reality.** B608 (SQL), B105–B107
  (hardcoded password strings), and the `assert` check (B101) are the usual
  noise sources. Plan for a triage step, not a zero-findings gate. The common
  pattern is a **baseline**: run `bandit -r . -f json -o baseline.json` once,
  then `bandit -b baseline.json` in CI so only *new* findings fail the build.
- **`# nosec` is coarse.** A bare `# nosec` silences every check on that line.
  Prefer `# nosec B602` to suppress only the specific test, or Bandit will
  also hide unrelated future findings on the same line. Suppressions are
  invisible in reports unless you audit for them.
- **B101 and `-O`.** The `assert_used` check exists because Python's `-O` flag
  strips `assert` statements, so asserts used for enforcement vanish in
  optimized builds. Test code uses `assert` everywhere; most teams skip B101
  for their test directories.
- **Severity/confidence flags matter for signal.** Plain `bandit -r` reports
  everything; `-ll` (HIGH severity) or `-ii` (HIGH confidence) is what makes
  output actionable in CI. Tune before adopting, not after developers learn to
  ignore the job.
- **Scope is Python only, and only what the AST sees.** It does not analyze
  dependencies, templates, generated code, or `exec`'d strings. It is not an
  SCA/dependency scanner — pair it with pip-audit or Safety for CVEs in
  packages.
- **Ruff overlaps.** If you already run Ruff, its `S` rule group reimplements
  a large subset of Bandit's checks natively in Rust and runs far faster;
  many teams drop standalone Bandit once Ruff covers the checks they use[^3].

## When to Use / When Not

**Use when:**
- You want a zero-config first-pass security linter in CI for a Python
  codebase.
- You need to enforce simple house rules (no `shell=True`, no `yaml.load`, no
  MD5 for security) mechanically across many files.
- You want SARIF output feeding a code-scanning dashboard cheaply.

**Avoid (or supplement) when:**
- You need to know whether a finding is actually reachable from untrusted
  input — Bandit has no taint tracking; use Semgrep or CodeQL.
- You want dependency-vulnerability (CVE) coverage — that is SCA, a different
  tool class.
- You already run Ruff and its `S` rules cover your checks — the standalone
  tool may be redundant.

## Alternatives

- returntocorp/semgrep — pattern-based like Bandit but with dataflow/taint
  modes and a large community rule registry; use when you need cross-function
  reasoning and multi-language coverage.
- github/codeql — deep semantic taint analysis; use when soundness and low
  false-positive rate justify heavier setup and query authoring.
- astral-sh/ruff — reimplements much of Bandit's `S` rule set in Rust at a
  fraction of the runtime; use when you already lint with Ruff and want one
  tool.
- pyupio/safety — dependency CVE scanning, not source analysis; use alongside
  Bandit to cover known-vulnerable packages, not code patterns.
- pypa/pip-audit — audits installed packages against the PyPI advisory
  database; the maintained, no-signup option for the SCA gap Bandit leaves.

## History

| Version | Date | Notes |
|---------|------|-------|
| Origin | ~2014 | Started inside the OpenStack Security Project[^1]. |
| Rehome | 2018-04 | Moved to the PyCQA organization (current repository)[^4]. |
| 1.x | ongoing | Dropped Python 2 support; added TOML config, SARIF output, and signed container images on ghcr.io[^2]. |

## References

[^1]: Bandit README, "Overview" — "Bandit was originally developed within the OpenStack Security Project and later rehomed to PyCQA." https://github.com/PyCQA/bandit#overview
[^2]: Bandit documentation — plugin model, test/blacklist plugins, config, and container images. https://bandit.readthedocs.io/en/latest/
[^3]: Ruff rules — the `flake8-bandit` (`S`) rule group. https://docs.astral.sh/ruff/rules/#flake8-bandit-s
[^4]: PyCQA/bandit repository metadata — repository created 2018-04-26 under the PyCQA organization. https://github.com/PyCQA/bandit

## Tags

python, security, static-analysis, sast, linter, ast, security-scanner, cli, pycqa, code-quality
