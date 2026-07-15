# gotestyourself/gotestsum

> A wrapper around `go test -json` that prints human-readable output, re-runs flaky tests, and emits JUnit XML for CI.

[GitHub repo](https://github.com/gotestyourself/gotestsum) ·
[License: Apache-2.0](https://github.com/gotestyourself/gotestsum/blob/main/LICENSE)

## Overview

`gotestsum` is a command-line front end for Go's test runner. It does not run
tests itself — it shells out to `go test -json`, parses the resulting
[test2json](https://pkg.go.dev/cmd/test2json) event stream, and re-presents it
in formats tuned either for a human watching a terminal or for a CI system that
wants a machine-readable report[^1]. It is maintained under the `gotest.tools`
umbrella (installed via the import path `gotest.tools/gotestsum`) and has been in
continuous development since 2018[^2].

The defining tension is that it is a thin layer over an interface it does not
control. Everything `gotestsum` knows comes from the `test2json` stream that
`go test` produces; its power (rerun-fails, watch mode, slowest-test analysis,
JUnit conversion) is built entirely on parsing and re-invoking that stream. This
keeps it dependency-light and forward-compatible with new Go releases, but it
also means the tool inherits `go test`'s constraints: it cannot change how tests
are scheduled, cannot see anything the JSON stream omits, and has to treat
stderr specially because that is the only channel Go uses to report package build
failures.

Its audience is Go teams who have outgrown raw `go test ./...` — typically for
CI dashboards (JUnit XML) or flaky-suite triage (rerun + slowest). It is widely
adopted in large Go infrastructure projects; the README lists Kubernetes, Docker
(moby), etcd, HashiCorp Vault/Consul, Prometheus, and containerd among users[^1].

## Getting Started

```bash
# Install the binary
go install gotest.tools/gotestsum@latest
# Or run without installing
go run gotest.tools/gotestsum@latest
```

```bash
# Local development: line-per-test output with color
gotestsum --format testname

# CI: JUnit XML report plus a raw JSON log for later analysis
gotestsum --junitfile unit-tests.xml --jsonfile test-output.log -- -count=1 ./...

# Re-run failing (possibly flaky) tests instead of the whole suite
gotestsum --rerun-fails --packages="./..." -- -count=1
```

Arguments after `--` are passed through to `go test`. By default the command run
is `go test -json ./...`.

## Architecture / How It Works

The pipeline has three stages:

1. **Execution.** `gotestsum` builds a `go test -json ...` command (or an
   arbitrary script under `--raw-command`) and runs it as a subprocess. stdout
   is expected to be pure line-delimited test2json; stderr is watched separately
   because `go test` reports package build errors only on stderr.
2. **Parsing.** The `testjson` package
   ([`gotest.tools/gotestsum/testjson`](https://pkg.go.dev/gotest.tools/gotestsum/testjson))
   consumes each JSON event (`run`, `pass`, `fail`, `skip`, `output`) and
   accumulates an in-memory `Execution` model of packages and test cases, with
   captured output and elapsed times.
3. **Presentation.** Formatters (`dots`, `pkgname`, `testname`, `testdox`,
   `standard-quiet`, `standard-verbose`) are event handlers that render the
   stream as it arrives. `testdox` embeds
   [bitfield/gotestdox](https://github.com/bitfield/gotestdox) to turn test names
   into sentences. After the run, a summary block and a `DONE` line are printed.

The same accumulated `Execution` state drives the side outputs: `--junitfile`
serializes it to JUnit XML, `--jsonfile` writes the raw test2json back out for
reuse, and `gotestsum tool slowest` reads a JSON log to rank tests by elapsed
time. `--rerun-fails` re-invokes `go test` with a `-test.run` regex matching only
the failed tests, looping until each passes once or the attempt/failure limits
are hit. `--watch` uses [fsnotify](https://pkg.go.dev/github.com/fsnotify/fsnotify)
to watch directories containing `.go` files and re-runs the package that changed.

Because `-json` is a `go test` flag rather than a test-binary flag, running a
pre-compiled test binary requires piping it through `go tool test2json` yourself
under `--raw-command`.

## Production Notes

- **stderr means "error" under `--raw-command`.** Any bytes a custom command
  writes to stderr are treated as a build/execution failure, because that is how
  Go surfaces package build errors. Scripts that log to stderr will break the run
  unless you restructure them or use `--ignore-non-json-output-lines` (added in
  v1.7.0) to divert non-JSON stdout lines[^1].
- **`--rerun-fails` has argument-ordering footguns.** When you pass `go test`
  args after `--`, you must also give the package list via `--packages=` (not as
  a positional), otherwise the rerun cannot reconstruct which packages to target.
  Args destined for the test binary rather than `go test` must sit behind
  `-args`.
- **JUnit output wants the `go` binary.** Without `go` on `PATH`, gotestsum emits
  a "failed to lookup go version for junit xml" warning; set `GOVERSION` to
  suppress it. The `testsuite.name` / `testcase.classname` fields default to the
  full package path and often need `--junitfile-testsuite-name=short`
  (or `relative`) to satisfy a given CI system's grouping.
- **It cannot see beyond test2json.** Output written outside the framework (for
  example from `TestMain` before `m.Run()`, or raw prints that bypass the JSON
  encoder) may be attributed oddly or lost. gotestsum reports what `go test`
  reports and no more.
- **No scheduling control.** Parallelism, timeouts, and package selection are all
  `go test` concerns; gotestsum does not shard or distribute tests. For CI
  fan-out you still partition packages yourself.
- **Watch mode and large trees.** `--watch` registers filesystem watches per
  directory; very large repositories can hit OS watch limits (inotify on Linux),
  and multi-module repos need `--watch-chdir` so `go test` will run packages
  outside the main module.

## When to Use / When Not

**Use when:**
- Your CI needs JUnit XML from Go tests and you want reruns and slow-test
  reporting from one tool.
- You want readable, colorized local output (`testname`/`testdox`) instead of raw
  `go test` noise.
- You are fighting a flaky suite and want `--rerun-fails` plus `--watch`-driven
  iteration.

**Avoid when:**
- You only need a summary and are happy piping `go test -json` into a lighter
  parser — gotestsum's feature surface is more than you need.
- You need test sharding, distribution, or scheduling logic; that is not what
  this tool does.
- You want zero external tooling in CI and can post-process `-json` yourself.

## Alternatives

- mfridman/tparse — parses `go test -json` into summary tables via a pipe; use it
  when you want a compact pass/fail summary and nothing else.
- jstemmer/go-junit-report — converts `go test` output to JUnit XML; use it when
  JUnit conversion is the only feature you need in CI.
- bitfield/gotestdox — sentence-style test output; gotestsum embeds it as the
  `testdox` format, so use it standalone only if that is all you want.
- kyoh86/richgo — colorized/decorated `go test` output; use it for local
  readability without reruns, watch, or JUnit.
- rakyll/gotest — minimal wrapper that colorizes standard `go test` output; use
  it when you want the plain format with color and no other behavior.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.1 | 2018-05-02 | Initial release. |
| v0.3.0 | 2018-05-05 | Early formats and JUnit output. |
| v0.4.0 | 2019-10-27 | Feature release. |
| v0.5.0 | 2020-06-17 | Feature release. |
| v0.6.0 | 2020-10-25 | Last of the 0.x line. |
| v1.6.1 | 2021-01-16 | Jumped to the 1.x line; watch-mode `r`/`d` keys[^1]. |
| v1.7.0 | 2021-07-23 | `--ignore-non-json-output-lines`; watch `a`/`l` keys[^1]. |
| v1.8.0 | 2022-04-09 | Feature release (watch `u` key in v1.8.1)[^1]. |
| v1.9.0 | 2023-01-14 | Feature release. |
| v1.10.0 | 2023-04-07 | Feature release. |
| v1.11.0 | 2023-09-16 | Feature release. |
| v1.12.0 | 2024-05-30 | Feature release. |
| v1.13.0 | 2025-09-11 | Latest tagged release[^2]. |

## References

[^1]: gotestsum README — features, formats, `--rerun-fails`/`--watch`/`--raw-command` behavior, and user list. https://github.com/gotestyourself/gotestsum/blob/main/README.md
[^2]: gotestsum releases (tags and dates). https://github.com/gotestyourself/gotestsum/releases

## Tags

go, golang, testing, test-runner, ci, junit-xml, cli, go-test, developer-tools, test-reporting
