# babashka/babashka

> A native, fast-starting Clojure interpreter for scripting — Clojure where you would otherwise reach for Bash.

[GitHub repo](https://github.com/babashka/babashka) ·
[Official website](https://babashka.org) ·
[License: EPL-1.0](https://github.com/babashka/babashka/blob/master/LICENSE)

## Overview

Babashka (`bb`) is a scripting runtime for Clojure, created by Michiel Borkent (borkdude) and first released in 2019[^1]. Ordinary Clojure runs on the JVM, whose startup cost — several hundred milliseconds to over a second before your code executes — makes it impractical for the small glue scripts, git hooks, and CI steps that shells are usually used for. Babashka closes that gap: it is a self-contained native binary that starts in single-digit milliseconds and ships with a curated set of Clojure and Java functionality built in. Its stated goal is to be used "in places where you would be using bash otherwise," not to replace the JVM for heavy work.

The runtime is built from two pieces: **SCI** (the Small Clojure Interpreter, also by borkdude), which interprets a substantial subset of Clojure form-by-form rather than compiling to JVM bytecode, and **GraalVM native-image**, which ahead-of-time compiles the whole thing into a standalone executable with no JVM dependency[^2]. That combination is the source of both babashka's advantage and its constraints.

The defining tradeoff: because GraalVM native-image uses a closed-world assumption, babashka embeds a *fixed, pre-selected* set of Java classes and cannot load new ones at runtime, and because SCI interprets rather than compiles, CPU-bound loops run slower than on the JVM. Babashka is therefore excellent for I/O-bound, short-lived, startup-sensitive scripts and a poor fit for long-running compute. The maintainer is explicit about this: if a script runs more than a few seconds or is loop-heavy, JVM Clojure is the better tool[^3].

## Getting Started

```bash
# macOS / Linux via Homebrew
brew install borkdude/brew/babashka

# or the portable installer
curl -sLO https://raw.githubusercontent.com/babashka/babashka/master/install
chmod +x install && ./install
```

A script is just a file with a shebang; no project or classpath setup is needed:

```clojure
#!/usr/bin/env bb
(require '[babashka.fs :as fs])

;; print the first three subdirectories of the cwd
(->> (fs/list-dir ".")
     (filter fs/directory?)
     (map (comp str fs/file-name))
     (take 3)
     (run! println))
```

Babashka also doubles as a task runner. A `bb.edn` in the project root defines tasks that replace an ad-hoc Makefile:

```clojure
;; bb.edn
{:paths ["src"]
 :deps  {}
 :tasks {:requires ([babashka.fs :as fs])
         clean {:doc "Remove build output"
                :task (fs/delete-tree "target")}
         build {:depends [clean]
                :task (shell "npm" "run" "build")}}}
```

Run with `bb clean`, `bb build`, or `bb tasks` to list them.

## Architecture / How It Works

Babashka does not compile your script. SCI walks each form and evaluates it against an interpreter that implements a large subset of `clojure.core`, `java.lang`, and a selection of common libraries. The interpreter itself, plus those libraries, are compiled once by GraalVM into the `bb` binary. Consequences that follow directly from this design:

- **A fixed Java surface.** Only classes that were included at build time are available (`System`, `File`, `java.time.*`, `java.nio.*`, `java.util.*`, and many more). You cannot `import` an arbitrary JVM class at runtime, because native-image did not compile it in. This is the single most common source of "works in Clojure, fails in babashka" surprises.
- **Batteries included.** Built-in namespaces cover the scripting essentials: `babashka.fs` (files), `babashka.process` / `clojure.java.shell` (subprocesses), `clojure.tools.cli` and `babashka.cli` (arg parsing), `cheshire`/`clojure.data.json` (JSON), `clojure.data.csv`, `clojure.java.io`, an HTTP client, and an nREPL server. The full list lives in the babashka book[^4].
- **Interpreter semantics.** `defprotocol` and `defrecord` are implemented on top of multimethods and plain maps rather than generating real Java classes; `deftype`/`definterface` and unboxed math are unsupported; `reify` handles one class at a time; and `clojure.core.async/go` is not implemented — it currently maps to `thread` for source compatibility[^3].
- **External dependencies.** `bb.edn`'s `:deps` can pull pure-Clojure libraries from Maven/Clojars, but only libraries whose code the interpreter can actually run — no macros or Java classes that require the missing compilation surface.
- **Pods.** For functionality that genuinely needs native or JVM code (SQLite, cryptography, etc.), babashka talks to out-of-process programs called *pods* over a defined protocol, exposing them as ordinary Clojure namespaces[^5].

## Production Notes

- **Startup is the whole point.** `bb` typically starts in a few milliseconds, which is why it is well suited to git hooks, `Makefile` replacement, CI steps, and CLIs invoked in a loop. Measure it before assuming; version managers that shim the binary (notably `asdf`) add noticeable startup overhead — the docs recommend `mise` over `asdf` for that reason.
- **Not for hot loops.** Interpretation overhead is real. Numeric or tight-loop workloads that would be trivial on the JVM can dominate a babashka script's runtime. Profile, and move genuinely compute-heavy work to JVM Clojure or a pod.
- **The "missing class" wall.** A library that transitively touches a Java class not baked into the binary will fail at runtime, not install time. Check the [compatible projects list](https://github.com/babashka/babashka/blob/master/doc/projects.md) and the built-in namespace list before committing to a dependency.
- **AWS Lambda.** The Lambda runtime does not support signals, so babashka must be told to disable its SIGINT/SIGPIPE handlers via `BABASHKA_DISABLE_SIGNAL_HANDLERS=true`.
- **Static binaries for musl.** On Alpine and other musl-libc systems, use the static Linux binary from GitHub Releases rather than the default dynamically linked one. WSL1 users are also directed to the `--static` install to avoid a reported BSOD.
- **Stability contract is partial.** The maintainer considers `clojure.core` and `java.lang` behavior stable, but explicitly reserves the right to change other parts. Read `CHANGELOG.md` before upgrading rather than assuming semver-strict compatibility.

## When to Use / When Not

**Use when:**
- You need Clojure's data handling and expressiveness for a script but cannot afford JVM startup.
- You are writing CLI tools, git hooks, CI/CD steps, or a task runner and want a single portable binary with no runtime dependency.
- The workload is I/O-bound and short-lived (shelling out, HTTP, file munging, JSON/CSV wrangling).
- Your team already knows Clojure and wants to stop context-switching into Bash.

**Avoid when:**
- The script is CPU-bound or long-running — JVM Clojure's throughput will beat babashka's startup savings.
- You depend on JVM libraries that need arbitrary runtime class loading, reflection, `deftype`, or macros the interpreter cannot run.
- You need `core.async` `go` blocks or other unsupported constructs with their real semantics.
- The task is trivial enough that plain Bash or a one-line shell pipeline is genuinely simpler.

## Alternatives

- clojure/clojure — full JVM Clojure; use it when you need compiled performance, complete Java interop, or long-running processes and can absorb the startup cost.
- babashka/nbb — the same author's Node.js-hosted Clojure scripting runtime; use it when you need npm/Node libraries instead of the JVM ecosystem.
- candid82/joker — a Go-based, fast-starting Clojure dialect and linter; lighter still, but implements a smaller subset of the language.
- planck-repl/planck — self-contained ClojureScript scripting on JavaScriptCore; use it when a ClojureScript-flavored, browser-adjacent surface fits better.
- GNU Bash / Python — reach for these when the job is trivial glue and pulling in a Clojure runtime is not worth it.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial release | 2019 | First public babashka, built on SCI + GraalVM native-image[^1]. |
| 0.2.x | 2020 | `bb.edn`, expanding built-in namespaces, `babashka.fs`. |
| 0.4.x | 2021 | Task runner (`bb tasks`), broader library compatibility. |
| 1.0.0 | 2023 | First 1.0; `clojure.core`/`java.lang` surface declared stable[^3]. |

## References

[^1]: Babashka project site and introduction. https://babashka.org
[^2]: Michiel Borkent, "Babashka: how GraalVM helped create a fast-starting scripting environment for Clojure." https://medium.com/graalvm/babashka-how-graalvm-helped-create-a-fast-starting-scripting-environment-for-clojure-b0fcc38b0746
[^3]: babashka README — "Setting expectations", "Status", and "Differences with Clojure." https://github.com/babashka/babashka
[^4]: Babashka book — built-in namespaces. https://book.babashka.org/#built-in-namespaces
[^5]: babashka pods documentation. https://github.com/babashka/pods

## Tags

clojure, scripting, graalvm, native-image, cli, shell-scripting, sci, interpreter, task-runner, jvm-alternative
