# unisonweb/unison

> A statically-typed functional language where code is content-addressed:
> definitions are identified by a hash of their syntax tree, not by files.

[GitHub repo](https://github.com/unisonweb/unison) ·
[Official website](https://www.unison-lang.org) ·
[License: MIT](https://github.com/unisonweb/unison/blob/trunk/LICENSE)

## Overview

Unison is a statically-typed functional language with type inference and an
algebraic effect system ("abilities"), developed by Unison Computing, a public
benefit corporation co-founded by Paul Chiusano and Rúnar Bjarnason (the
authors of *Functional Programming in Scala*). The compiler and tooling are
written in Haskell. After roughly a decade of development — the repo dates to
May 2015, with a first public alpha (M1) in 2019 — Unison reached 1.0 in
November 2025[^3].

The "big idea"[^1] is that a definition is identified by the hash of its AST,
and code is stored as ASTs in a database rather than as text files. Names are
metadata attached to hashes. The downstream consequences are the product:
there is no build step (definitions are typechecked once, ever), renames are
instant and non-breaking, deterministic test results are cached by hash, and
version control is semantic — whitespace, formatting, and import order cannot
cause merge conflicts. Because a definition's dependencies are pinned by hash,
computations can also be serialized and shipped to other machines, which is
the basis of Unison Cloud, the company's commercial distributed-computing
platform[^4].

The defining tradeoff: abandoning plain-text source as the source of truth
means abandoning the entire conventional toolchain. Git, file-based editors,
grep, and standard CI pipelines do not operate on a Unison codebase; the
Unison Codebase Manager (UCM) replaces version control, build tool, package
manager, and REPL in one. The ecosystem is correspondingly small — 6.7k stars
and ~300 forks after eleven years is a research-grade following, not
mainstream adoption.

## Getting Started

```bash
brew install unisonweb/unison/unison-language   # or grab a GitHub release
ucm    # starts the codebase manager + local web UI
```

You write code in a *scratch file* that UCM watches; watch expressions
(`>`) evaluate on save:

```haskell
-- scratch.u
factorial : Nat -> Nat
factorial n = product (range 1 (n + 1))

> factorial 5
```

Then, inside UCM: `add` typechecks and stores the definitions, `run main`
executes a program, `push @you/myproject` publishes to Unison Share.

## Architecture / How It Works

**Codebase.** A SQLite database of ASTs keyed by content hash, stored under
`.unison/`. Source text is a projection: when you `view` a definition, UCM
pretty-prints the stored AST using your current names. Two definitions that
differ only in variable names or formatting hash identically.

**UCM workflow.** You never edit "the project" directly. You `edit` a
definition (UCM writes it into the scratch file), change it, and `update`,
which re-hashes it and propagates the change to dependents; anything that no
longer typechecks lands back in the scratch file for you to fix.

**Abilities.** Unison's algebraic effect system, influenced by the Frank
language. Effects appear in types (`Nat ->{IO, Exception} ()`) and get their
semantics from handlers, so mocking and structured concurrency fall out of
the type system. The second-biggest learning-curve item after the codebase
model.

**Runtime.** An interpreter inside UCM for interactive use, plus a native
code path (`compile`) that produces standalone executables through a
Racket-based runtime. Interactive evaluation is fast to start because nothing
needs building; raw compute throughput is not the design center.

**Dependencies.** Libraries live under a `lib` namespace, vendored by hash
from Unison Share[^5] — there is no version solver and no diamond-dependency
problem, because two versions of a library are simply different hashes. The
`upgrade` command rewrites your code to a new library version. Docs are typed
values checked alongside code, so examples in documentation cannot go stale.

**Editor/agent story.** An LSP server runs over UCM (you still edit only
scratch files), and recent UCM versions ship an MCP server so AI agents can
query and manipulate the codebase directly[^2].

## Production Notes

- **Tooling lock-in is total.** No git diff/blame/bisect on the codebase; the
  SQLite database under `.unison/` is the artifact to back up. CI runs via
  UCM *transcripts* — markdown files of UCM commands with recorded output —
  which is workable but unlike anything your existing pipeline does.
- **UCM version upgrades migrate the codebase schema**, and migrations are
  effectively one-way. Snapshot `.unison/` before upgrading UCM.
- **Performance:** fine for IO-bound services and glue; do not choose Unison
  for CPU-bound hot loops. Native compilation narrows the gap but the
  numerics/data ecosystem is thin.
- **No general-purpose FFI.** Interop is what the builtins and ecosystem
  provide; you cannot casually bind a C library the way you would in Haskell
  or Rust. Check Unison Share for what exists before committing.
- **Single-vendor risk.** Development is driven almost entirely by Unison
  Computing, whose revenue is Unison Cloud. The project is genuinely active —
  releases roughly monthly through 2025–2026, last push July 2026 — but the
  bus factor is a company, not a foundation.
- **Onboarding:** expect weeks, not days. The codebase model, abilities, and
  UCM workflow each invalidate file-based habits; ~1,300 open issues also
  means you will hit tooling rough edges.

## When to Use / When Not

**Use when:**
- You want to experience where language tooling could go: no builds, cached
  tests, semantic merge, hash-pinned dependencies.
- You are building distributed systems and the pair-with-Unison-Cloud model
  (typed remote computation, no serialization boilerplate) fits.
- You value an effect system that makes side effects explicit and testable.
- Greenfield side projects and research, where lock-in costs little.

**Avoid when:**
- You need ecosystem depth — libraries, FFI, hiring pool, Stack Overflow
  answers. Unison has none of these at scale.
- Your org's compliance/CI story assumes text files in git.
- CPU-bound performance matters.
- You need multi-decade stability guarantees from a foundation-governed
  language; Unison is one PBC's ongoing bet.

## Alternatives

- roc-lang/roc — new functional language focused on speed and a conventional
  file-based toolchain; use it when you want fresh FP without tooling lock-in.
- gleam-lang/gleam — typed functional on the BEAM with ordinary files and
  packages; use it for production-ready typed FP with a growing ecosystem.
- koka-lang/koka — use it if algebraic effects are the part of Unison you
  want, in a conventional research compiler.
- darklang/dark — the closest philosophical cousin (code-in-database,
  deployless); use it for integrated backend-as-language rather than a
  general-purpose language.
- ghc/ghc — Haskell; use it when you want Unison's type-system lineage with
  a mature ecosystem and real FFI.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2015-05 | Repository created; long research phase begins. |
| M1 | 2019 | First public alpha of the language and UCM. |
| 0.5.x | 2023–2025 | Rolling pre-1.0 releases: native runtime, semantic `merge`, LSP maturation (series ends at 0.5.50, 2025-11). |
| 1.0.0 | 2025-11-25 | First stable release[^3]. |
| 1.1.0 | 2026-01-28 | Post-1.0 cadence begins (~monthly). |
| 1.2.0 | 2026-04-17 | |
| 1.3.0 | 2026-05-20 | Latest stable at time of writing. |

## References

[^1]: Unison docs, "The big idea". https://www.unison-lang.org/docs/the-big-idea/
[^2]: unisonweb/unison README (LSP, MCP server, codebase server). https://github.com/unisonweb/unison#readme
[^3]: Unison 1.0.0 release — 2025-11-25. https://github.com/unisonweb/unison/releases/tag/release%2F1.0.0
[^4]: Unison Cloud. https://www.unison.cloud
[^5]: Unison Share — ecosystem/library host. https://share.unison-lang.org

## Tags

programming-language, functional, haskell, content-addressed, algebraic-effects, type-inference, distributed-systems, developer-tools, compiler
