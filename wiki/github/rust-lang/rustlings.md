# rust-lang/rustlings

> Small compile-error exercises that teach you to read and write Rust by fixing broken code.

[GitHub repo](https://github.com/rust-lang/rustlings) ·
[Official website](https://rustlings.rust-lang.org) ·
[License: MIT](https://github.com/rust-lang/rustlings/blob/main/LICENSE)

## Overview

Rustlings is a collection of small Rust exercises, each a source file that does
not compile or does not pass its tests. Your job is to make it build and pass.
It is explicitly positioned as a companion to *The Rust Programming Language*
book — read a chapter, then do the matching exercises[^1]. It is not a
standalone curriculum and assumes you are reading prose elsewhere.

The defining design choice is **learning by fixing, not by writing from scratch**.
Every exercise hands you almost-correct code with a specific defect: a missing
`mut`, a moved value used after move, a type mismatch, an unhandled `Result`.
This narrows the search space to one concept at a time and makes the compiler
the teacher — you iterate against `rustc`'s error messages until they go quiet.
The tradeoff is that Rustlings trains you to *recognize and repair* idioms far
better than it trains you to *design* programs on a blank page; it is deliberately
not a project-based course.

Rustlings is a `rust-lang` org project (originally created by Carol Nichols in
2015) and tracks the current stable toolchain, so exercises assume a recent
`rustc`. It underwent a full rewrite for the 6.x line, which replaced the older
`rustlings watch` workflow and the `// I AM NOT DONE` marker convention with a
single interactive terminal UI[^2].

## Getting Started

Rustlings requires a working Rust toolchain (install via `rustup`). Then:

```bash
cargo install rustlings
rustlings init
cd rustlings
rustlings          # launches the interactive watch mode
```

`rustlings init` scaffolds an `exercises/` directory and a state file into the
current folder. Running `rustlings` with no arguments drops you into the first
unsolved exercise and re-checks on every save. A typical exercise looks like:

```rust
// variables2.rs
fn main() {
    let x;              // error: type must be known / value must be assigned
    if x == 10 {
        println!("x is ten!");
    } else {
        println!("x is not ten!");
    }
}
```

The fix — `let x = 10;` — is minimal by design. Other subcommands: `rustlings run
<name>` runs one exercise, `rustlings hint <name>` prints a graded hint, and
`rustlings reset <name>` restores an exercise to its broken starting state.

## Architecture / How It Works

Rustlings is itself a Rust binary (a Cargo crate) that manages and executes a
tree of exercise files. The core loop is a file watcher plus a runner:

1. **Exercise manifest** — an embedded list (historically `info.toml`) defines
   each exercise: its path, its mode (`compile`, `test`, or `clippy`), and its
   hint text.
2. **Mode dispatch** — `compile` exercises must `rustc`-build; `test` exercises
   must pass an embedded `#[test]`; `clippy` exercises must pass with zero
   `cargo clippy` warnings.
3. **Watch + advance** — on save, the changed exercise is recompiled/tested. On
   success the UI advances; progress is persisted in a state file so you can
   quit and resume.

Exercises are grouped by topic in rough book order: `variables`, `functions`,
`if`, `primitive_types`, `vecs`, `move_semantics`, `structs`, `enums`,
`strings`, `modules`, `hashmaps`, `options`, `error_handling`, `generics`,
`traits`, `tests`, `lifetimes`, `iterators`, `smart_pointers`, `threads`,
`macros`, `clippy`, `conversions`, plus interspersed `quiz` checkpoints that
combine several concepts.

The 6.x rewrite is the most consequential architectural event. Earlier versions
detected "unstarted" exercises with a literal `// I AM NOT DONE` comment you
deleted as you worked, and shelled out to `cargo`. The rewrite dropped the
marker, moved to its own custom terminal UI, and made state tracking explicit
rather than comment-driven[^2]. Course material written against 5.x (blog posts,
videos) is partially stale as a result.

## Production Notes

Rustlings is a learning tool, so "production" here means the classroom-and-
self-study operating surface — where it bites in practice.

- **Toolchain drift.** Because exercises target current stable Rust, an old
  `rustc` can produce errors that don't match the intended lesson. Run `rustup
  update` before starting. Conversely, a *newer* compiler occasionally changes a
  lint or error message so a hint reads slightly off from what you see.
- **Version-specific instructions everywhere.** Third-party tutorials frequently
  reference `rustlings watch` or deleting `I AM NOT DONE`. On a 6.x install
  those commands/markers do not exist. Trust the built-in `rustlings hint` and
  the official website over dated write-ups.
- **`clippy` exercises need the component.** The clippy-mode exercises fail
  confusingly if `clippy` isn't installed (`rustup component add clippy`).
- **It won't teach architecture.** The single-file, fix-the-defect format means
  learners can finish all exercises and still struggle to structure a real
  multi-module crate, manage dependencies, or design ownership across a codebase.
  Pair it with an actual project.
- **Hints can be a crutch.** `rustlings hint` often walks most of the way to the
  answer. For borrow-checker and lifetime exercises specifically, reading the
  raw compiler error first is the higher-value exercise.
- **Solutions are available.** The repo ships reference solutions; the honor
  system is the only thing stopping you from copying them, which defeats the
  purpose for the hardest exercises (lifetimes, iterators, smart pointers).

## When to Use / When Not

**Use when:**
- You are reading the Rust book and want hands-on reps per chapter.
- You learn well from compiler feedback and small, bounded puzzles.
- You want to drill ownership, borrowing, and error-handling idioms until they
  are automatic.
- You are onboarding a team to Rust and want a consistent, low-setup warm-up.

**Avoid when:**
- You have never programmed before — Rustlings assumes general programming
  fluency and moves fast.
- You want to learn by building something real; a guided project course fits
  better.
- You need depth on async, macros, or systems-level `unsafe` — coverage of these
  is thin to absent.
- You want mentored feedback rather than a compiler and static hints.

## Alternatives

- rust-lang/book — the canonical prose course Rustlings is designed to accompany; use this as the primary text and Rustlings as the drills.
- rust-lang/rust-by-example — runnable annotated examples; use when you prefer reading-and-running over fixing broken code.
- google/comprehensive-rust — Google's multi-day slide-and-exercise course; use for instructor-led or classroom settings.
- mainmatter/100-exercises-to-learn-rust — fix-the-code exercises built around one evolving project; use when you want more continuity and depth than Rustlings' isolated files.
- exercism/rust — mentored track with human feedback; use when you want a person to review your solutions.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2015-09 | Repo created by Carol Nichols; early exercise set.[^3] |
| 4.x–5.x | 2021–2023 | `rustlings watch`, `// I AM NOT DONE` markers, `cargo`-driven runner. |
| 6.0 | 2024-11 | Full rewrite: custom interactive TUI, `rustlings init`, markers removed.[^2] |

## References

[^1]: Rustlings README and website — recommended in parallel with *The Rust Programming Language* book. https://rustlings.rust-lang.org
[^2]: Rustlings 6.x rewrite — new interactive UI and `rustlings init` workflow replacing `watch` and the `I AM NOT DONE` marker. https://github.com/rust-lang/rustlings/releases
[^3]: GitHub repository metadata — `rust-lang/rustlings`, created 2015-09-15. https://github.com/rust-lang/rustlings

## Tags

rust, learning, exercises, beginner-friendly, education, cli, compiler-driven, self-study, rust-lang, teaching
