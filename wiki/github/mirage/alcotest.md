# mirage/alcotest

> The de facto standard OCaml unit test framework — small API, colourful
> output, and a Cmdliner-powered CLI baked into every test binary.

[GitHub repo](https://github.com/mirage/alcotest) ·
[API docs](https://mirage.github.io/alcotest/alcotest/Alcotest/index.html) ·
License: ISC

## Overview

Alcotest is a lightweight unit test framework for OCaml, started in 2013 by
Thomas Gazagnaire inside the MirageOS organization[^1] and since adopted far
beyond it — it is the most common test dependency in the opam ecosystem and
the default answer to "how do I test OCaml code"[^5]. Its 514 GitHub stars
badly undercount that position: OCaml libraries accrue stars slowly, and
Alcotest's reach is better measured by its ubiquity in `dune-project`
`depends` stanzas across the ecosystem.

The design center is deliberate minimalism: a `TESTABLE` module type (a
pretty-printer plus an equality), a `check` function that asserts with typed
combinators (`Alcotest.(check (list int))`), and a `run` function that turns a
list of named test cases into a self-contained executable. Failure output is
the selling point — quiet on success, and on failure a coloured
expected/actual diff rendered by the type's own printer, with full logs
captured to disk under `_build/_tests`[^2].

The defining tradeoff is that Alcotest is a test *runner and assertion
library*, nothing more. There is no mocking, no fixtures, no setup/teardown
protocol, no property-based generation, and no test discovery — you enumerate
every case by hand in the `run` call. In OCaml practice this is treated as a
feature (the type system does much of what mocks do elsewhere), and gaps are
filled by composing with c-cube/qcheck or janestreet/ppx_expect rather than by
Alcotest growing features.

## Getting Started

```bash
opam install alcotest
# or in dune-project:  (depends ... (alcotest :with-test))
```

```ocaml
(* test/test_utils.ml *)
let test_lowercase () =
  Alcotest.(check string) "same string" "hello!"
    (String.lowercase_ascii "hELLO!")

let test_concat () =
  Alcotest.(check (list int)) "same lists" [1; 2; 3]
    (List.append [1] [2; 3])

let () =
  Alcotest.run "Utils" [
    "strings", [ Alcotest.test_case "lowercase" `Quick test_lowercase ];
    "lists",   [ Alcotest.test_case "concat"    `Quick test_concat ];
  ]
```

```dune
(test (name test_utils) (libraries alcotest))
```

Run with `dune runtest`. The built binary is itself a CLI: `test_utils.exe
--help`, `--verbose`, `-q` (quick tests only), or a regex to filter suites.

## Architecture / How It Works

The core abstraction is `TESTABLE`: a first-class module packaging `pp` (an
`Fmt` printer) and `equal` for some type. Alcotest ships combinators for base
types (`int`, `string`, `float` with epsilon, `option`, `result`, `list`,
`pair`, ...) and a `testable` function to build one from any printer +
equality. `check t msg expected actual` raises a located failure on mismatch,
formatted via `pp` — this is why Alcotest failures are readable where
polymorphic-compare assertions are not.

`run` compiles the suite list into a Cmdliner-based command-line program[^3].
Test selection, verbosity, colour, and JUnit-ish log placement are all CLI
concerns of *your* binary, not of an external runner — the opposite of
pytest-style discovery. `run_with_args` extends this by threading a
user-supplied Cmdliner term into every test function (optional arguments
only; positional arguments are unsupported).

Since the 1.0.0 rewrite (2020)[^4], the framework is split into an
`alcotest.engine` core functorized over a platform and concurrency monad,
with concrete backends published as separate opam packages: `alcotest`
(Unix, identity monad), `alcotest-lwt`, `alcotest-async`, `alcotest-mirage`,
and JavaScript support via js_of_ocaml. `Alcotest_lwt` accepts
`unit -> unit Lwt.t` cases, runs `Lwt_main.run` for you, converts async
exceptions into test failures instead of process exits, and hands each case
an `Lwt_switch` for resource cleanup — the closest thing Alcotest has to
teardown.

## Production Notes

- **Filtering is by suite name and index, not case name.** You can select
  `'string-.*'` or `'strings' '1..3'`, but never "the test called
  lowercase"[^2]. Saved CI invocations that pin numeric ranges silently
  rot when someone inserts or reorders a test case.
- **Output capture hides your prints.** Stdout/stderr of passing tests go to
  log files under `_build/_tests`, not the console. Debugging with
  `print_endline` requires `--verbose` or digging into the log directory;
  only the first failing test's log is shown by default (`--show-errors`
  prints all).
- **The CLI is Cmdliner's, including env vars.** `ALCOTEST_COLOR`,
  `ALCOTEST_VERBOSE`, and `ALCOTEST_QUICK_TESTS` control behaviour in CI
  where flags are awkward to pass through `dune runtest`.
- **No intra-suite parallelism.** Cases in one binary run sequentially;
  parallelism in practice comes from dune running multiple test executables
  concurrently. Long suites are usually split into several `(test ...)`
  stanzas rather than sped up in-process.
- **`` `Slow`` is opt-out, not opt-in.** Slow tests run by default and are
  suppressed with `-q` — teams expecting the reverse get surprised CI times.
- **Custom `TESTABLE` boilerplate.** Every domain type you assert on needs a
  printer and equality; with ppx_deriving's `show`/`eq` this is one line,
  without it it is the main day-to-day cost of the library.
- Maintenance is steady rather than fast: MirageOS-org stewardship, a push as
  recent as June 2026, 61 open issues/PRs, roughly yearly releases — for a
  test framework, a stability signal rather than a red flag.

## When to Use / When Not

**Use when:**
- You are writing OCaml and want the ecosystem-default test harness that
  dune, opam's `with-test`, and CI templates already assume.
- You want readable coloured diffs from typed assertions with minimal API
  surface to learn.
- You need Lwt or Async test cases with exception containment and cleanup
  hooks (`alcotest-lwt` / `alcotest-async`).

**Avoid when:**
- You want property-based or fuzz testing — use c-cube/qcheck or
  stedolan/crowbar (they integrate with Alcotest but the generation engine
  is theirs).
- You want inline, discovery-style tests colocated with source — Jane
  Street's ppx_inline_test/ppx_expect model suits monorepo workflows better.
- You need rich fixtures, mocking, parameterized matrices, or parallel
  in-process execution as framework features; Alcotest will not grow them.

## Alternatives

- gildor478/ounit — use instead for xUnit-style structure and assertions if
  migrating from JUnit-family habits; noisier output, older API.
- c-cube/qcheck — use instead (or alongside) when properties over random
  inputs beat hand-picked examples.
- janestreet/ppx_expect — use instead for snapshot/expect tests where the
  expected output lives inline and is auto-promoted.
- janestreet/ppx_inline_test — use instead for `let%test` cases embedded in
  source files with build-time discovery.
- stedolan/crowbar — use instead for AFL-driven, coverage-guided property
  testing on the OCaml compiler's AFL instrumentation.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2013–2019 | Grew inside MirageOS as the org's test harness; repo created 2013-12[^1]. |
| 1.0.0 | 2020 | Major internal rewrite: `alcotest.engine` split from platform backends (`alcotest-lwt`, `alcotest-async`, `alcotest-mirage`), improved expected/actual diffing[^4]. |
| 1.x | 2020–present | Incremental releases: OCaml 5 compatibility, js_of_ocaml support, output and CLI refinements. Active as of mid-2026. |

## References

[^1]: GitHub repository (created 2013-12-12). https://github.com/mirage/alcotest
[^2]: Alcotest README — output capture, test selection language. https://github.com/mirage/alcotest#readme
[^3]: Cmdliner — the CLI library every Alcotest binary embeds. https://erratique.ch/software/cmdliner
[^4]: Alcotest 1.0.0 release notes. https://github.com/mirage/alcotest/releases/tag/1.0.0
[^5]: Alcotest on opam. https://opam.ocaml.org/packages/alcotest/

## Tags

ocaml, testing, test-framework, unit-testing, tap, colored-output, cmdliner, developer-tools, library
