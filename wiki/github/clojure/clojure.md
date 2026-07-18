# clojure/clojure

> A Lisp for the JVM built around immutable data structures — designed by one person, changed slowly on purpose.

[GitHub repo](https://github.com/clojure/clojure) ·
[Official website](https://clojure.org) ·
[License: EPL-1.0](https://github.com/clojure/clojure/blob/master/epl-v10.html)

## Overview

Clojure is a dynamically typed functional Lisp hosted on the JVM, designed by Rich Hickey and released publicly in 2007, with 1.0 in May 2009[^1]. Its core bet is that most program complexity comes from mutable state, and that the fix is immutable persistent data structures (maps, vectors, sets, lists) as the default, with mutation quarantined behind explicit reference types (atoms, refs with STM, agents). The second bet is being a *hosted* language: Clojure does not hide the JVM — Java interop is idiomatic, the standard library leans on `java.util` and friends, and the GitHub language stat reads "Java" because the compiler, runtime (`RT.java`), and the persistent collections are implemented in Java, with the standard library bootstrapped in `core.clj`.

The repo itself is unusual among major languages. The star count (~10.9k) understates the language's industrial footprint (Nubank — which employs Hickey and the core team — Walmart, Cisco, and a durable consulting economy) because the repo is not the community's front door: GitHub issues are disabled (`open_issues_count: 0`), pull requests are not accepted, and contributions flow through ask.clojure.org and JIRA with a signed contributor agreement[^2]. Development is deliberately conservative — Hickey's stated policy is accretion over breakage ("don't break the ecosystem")[^3], and code written against 1.2 in 2010 generally still runs on 1.12. The repo remains actively maintained (last push 2026-07) but with a commit cadence that would look dormant for a JavaScript framework; here it is the design.

## Getting Started

```bash
# macOS
brew install clojure/tools/clojure
# Linux: see https://clojure.org/guides/install_clojure
clj    # starts a REPL
```

A project is a `deps.edn` file plus source:

```clojure
;; deps.edn
{:deps {org.clojure/data.json {:mvn/version "2.5.1"}}}
```

```clojure
;; src/demo/core.clj
(ns demo.core
  (:require [clojure.string :as str]))

(defn top-words [text n]
  (->> (str/split (str/lower-case text) #"\W+")
       (frequencies)                 ; {"the" 4, "cat" 2, ...}
       (sort-by val >)
       (take n)))

;; in the REPL:
;; (top-words (slurp "war-and-peace.txt") 10)
```

The REPL is the primary development interface — Clojure workflows center on evaluating forms into a live running program from the editor (CIDER, Calva, Cursive), not on edit-compile-run cycles.

## Architecture / How It Works

Clojure compiles every form to JVM bytecode on the fly — there is no interpreter tier. `Compiler.java` (a single ~9k-line file) performs analysis and bytecode emission per top-level form via the ASM library; `fn` bodies become anonymous JVM classes. This gives Java-comparable steady-state performance for non-numeric code, at the cost of class-generation overhead at load time.

The persistent data structures are the technical heart: vectors and maps are Hash Array Mapped Tries (after Bagwell), 32-way branching trees where "modification" copies the path from root to changed leaf (~log32 n) and shares the rest[^4]. Structural sharing is what makes immutability-by-default affordable. `transient` versions allow locally-mutable batch construction that is frozen back into a persistent value.

Other load-bearing pieces:

- **Vars and dynamic binding** — top-level definitions are mutable indirection cells, which is what makes REPL redefinition work; `^:dynamic` vars give thread-local rebinding.
- **Seqs and laziness** — the `seq` abstraction unifies iteration over all collections; most of `clojure.core` (`map`, `filter`, etc.) returns lazy seqs. Transducers (1.7) later factored the transformation logic out of the seq machinery so it composes without intermediate allocation.
- **Protocols and multimethods** — polymorphism without classes. Protocols (1.2) compile to JVM interfaces for speed; multimethods dispatch on arbitrary functions for flexibility.
- **Concurrency primitives** — atoms (CAS), refs (software transactional memory), agents (async serialized updates). In practice atoms cover ~95% of real usage; the STM is intellectually notable but rarely load-bearing in production code.
- **spec (1.9)** — runtime data specification and generative testing, shipped as `spec.alpha` and still alpha-labeled years later; the community largely adopted metosin/malli instead.

## Production Notes

**Startup time is the canonical complaint.** JVM boot plus loading `clojure.core` costs roughly a second before your code runs; a Leiningen-era toolchain could add several more. Fine for long-lived servers, painful for CLIs and serverless. Mitigations: AOT compilation, class-data sharing, GraalVM native-image (with care around dynamic class loading), or sidestepping entirely with babashka for scripting.

**Reflection and boxing are silent performance cliffs.** Interop calls without type hints fall back to runtime reflection (10–100x slower); unhinted arithmetic boxes numbers. `(set! *warn-on-reflection* true)` and `*unchecked-math* :warn-on-boxed` belong in every performance-sensitive namespace — the defaults will not warn you.

**Lazy seq footguns.** Holding the head of a large lazy seq pins it all in memory; mixing laziness with side effects or scoped resources (`with-open` returning an unrealized seq reading a closed file) is a classic bug; chunked realization (32 at a time) surprises people who assumed one-at-a-time evaluation. Rule of thumb: laziness for pure transformation, `doseq`/`run!`/transducers when effects or resources are involved.

**Error messages and stack traces are Java-shaped.** A one-line mistake can produce a 40-frame trace through `clojure.lang.*` internals. 1.10 meaningfully improved macro-error reporting (spec-checked syntax, phase-tagged errors)[^5], but "NullPointerException from three abstraction layers down" remains part of the experience for beginners.

**Dynamic typing at scale is a real tradeoff.** Large codebases lean on spec or malli schemas at boundaries, heavy REPL-driven testing, and naming discipline ("maps of what?"). Teams that skip this pay for it in refactoring fear. There is no gradual-typing story with traction (core.typed exists but saw little adoption).

**Upgrades are boring — genuinely.** The last breaking release of note was 1.3 (2011), which changed numeric handling and dynamic-var defaults. Since then, bumping the Clojure version is typically a one-line change. The flip side: long-requested changes land slowly or never, and the core team's roadmap is not community-driven.

## When to Use / When Not

**Use when:**
- The domain is data transformation — ETL, APIs over heterogeneous data, rules engines — where immutable maps + `clojure.core` genuinely outperform class hierarchies for expressiveness.
- You need the JVM (libraries, ops maturity, performance) but want a higher-leverage language than Java on it.
- Interactive development matters: the REPL-into-live-system workflow is still ahead of most ecosystems.
- You value decade-scale stability of language and libraries over feature velocity.

**Avoid when:**
- Startup latency matters (CLIs, FaaS cold starts) and you can't adopt babashka/native-image workarounds.
- The team won't invest in the paradigm: Clojure's hiring pool is small, and half-adopted Clojure (Java written in parentheses) is worse than either language.
- You want static types as a primary correctness tool — pick a typed language rather than fighting for a bolt-on.
- You expect community-driven language evolution or GitHub-native contribution; the governance model is explicitly not that.

## Alternatives

- babashka/babashka — native-image Clojure interpreter; use it instead for scripts and CLIs where JVM startup is disqualifying.
- clojure/clojurescript — the same language compiled to JavaScript; use it when the target is the browser or Node.
- jank-lang/jank — Clojure dialect on LLVM with C++ interop, pre-1.0; watch it if you want Clojure semantics without the JVM.
- scala/scala3 — typed FP on the JVM; use it when you want static types and are willing to trade Clojure's simplicity for them.
- elixir-lang/elixir — dynamic FP with Lisp-ish tooling on the BEAM; use it when fault-tolerant concurrency (OTP) matters more than JVM access.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2009-05 | First stable release[^1]. |
| 1.2 | 2010-08 | Protocols, deftype/defrecord. |
| 1.3 | 2011-09 | Numeric handling overhaul; last significantly breaking release. |
| 1.5 | 2013-03 | Reducers, `cond->`/`as->` threading macros. |
| 1.7 | 2015-06 | Transducers; `.cljc` reader conditionals for Clojure/ClojureScript sharing. |
| 1.9 | 2017-12 | clojure.spec integration; official CLI + `deps.edn` era begins. |
| 1.10 | 2018-12 | Error-message overhaul, prepl; Java 8 minimum[^5]. |
| 1.11 | 2022-03 | Keyword args callable as maps, `clojure.math`. |
| 1.12 | 2024-09 | Method values, `add-lib` at the REPL, functional-interface interop[^6]. |

## References

[^1]: Rich Hickey, "A History of Clojure" — HOPL IV, 2020. https://clojure.org/about/history
[^2]: Clojure development / contribution workflow (no GitHub PRs; ask.clojure.org + contributor agreement). https://clojure.org/dev/dev
[^3]: Rich Hickey, "Spec-ulation" — Clojure/conj 2016 keynote on growth vs. breakage. https://www.youtube.com/watch?v=oyLBGkS5ICk
[^4]: Clojure reference — data structures (persistent, structural sharing). https://clojure.org/reference/data_structures
[^5]: Clojure changelog (`changes.md`, per-release detail). https://github.com/clojure/clojure/blob/master/changes.md
[^6]: Clojure 1.12 release announcement — 2024-09. https://clojure.org/news/2024/09/05/clojure-1-12-0

## Tags

clojure, lisp, jvm, functional-programming, dynamic-language, immutable-data, repl-driven, java-interop, programming-language, compiler
