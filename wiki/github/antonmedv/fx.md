# antonmedv/fx

> Terminal JSON viewer and processor that uses JavaScript as its query language instead of a bespoke DSL.

[GitHub repo](https://github.com/antonmedv/fx) ·
[Official website](https://fx.wtf) ·
[License: MIT](https://github.com/antonmedv/fx/blob/master/LICENSE)

## Overview

fx is a command-line tool for viewing and transforming JSON, written by Anton Medvedev (also author of `expr-lang/expr`, `walk`, and `countdown`)[^1]. It does two jobs in one binary: a full-screen interactive TUI for exploring a JSON document (fold/unfold nodes, regex search, dig into paths), and a non-interactive stream processor you pipe data through. The repository dates to 2018; as of 2026 it sits around 20.5k stars with steady but low-volume maintenance[^2].

The defining design choice is the query language: fx uses **JavaScript expressions** rather than a purpose-built DSL like jq's[^3]. `echo '{"n":1}' | fx '.n + 1'` works because the argument is evaluated as JavaScript with the input bound to the current value. This is the whole pitch — "no need to learn a DSL" — and also the whole tradeoff. JavaScript is familiar to most developers and needs no separate grammar, but it is heavier, less composable for long pipelines, and carries JS numeric and ordering semantics that differ from jq.

The second thing to understand about fx is that it ships as **two different programs under one name**. The original tool was a Node.js package on npm; since version 20 (2022) the flagship implementation is a complete rewrite in Go, distributed as a single self-contained binary[^4]. The interactive TUI viewer lives only in the Go binary — the npm/Deno JavaScript build operates in non-interactive processing mode only[^5]. Installing the wrong one gives you a strictly lesser tool with the same command name.

## Getting Started

```bash
# Go binary (interactive TUI + processing) — recommended
brew install fx
# or: go install github.com/antonmedv/fx@latest
# or: curl https://fx.wtf/install.sh | sh

# JavaScript build (processing only, no TUI)
npm install -g fx
```

```bash
# Interactive: explore a document in a full-screen viewer
curl -s https://api.github.com/repos/antonmedv/fx | fx

# Non-interactive: extract and transform with JavaScript
echo '{"users":[{"name":"Tom","admin":true},{"name":"Brad"}]}' \
  | fx '.users.filter(u => u.admin).map(u => u.name)'
# => [ "Tom" ]

# Chain multiple arguments — each is applied left to right
cat data.json | fx '.items' 'x => x.length'
```

## Architecture / How It Works

fx runs in one of two modes depending on whether stdout is a TTY and whether a reducer argument was given:

1. **Interactive viewer** (Go binary, no reducer, output is a terminal) — renders the parsed document as a collapsible tree. You navigate nodes, expand/collapse, run regex searches, and type a dot-path to jump into the structure. Long strings are wrapped or opened in a separate preview pane so a single huge value does not blow out the layout[^6].
2. **Stream processor** (any build, with reducer args, or piped output) — each command-line argument is a JavaScript expression or function. The current value is the input; the result of one argument becomes the input to the next, so multiple args form a pipeline. Output is printed as JSON (or raw, for strings).

Input handling is more permissive than strict JSON: fx accepts **JSON with comments and trailing commas**, and also reads **YAML and TOML** documents, normalizing them into the same tree[^6]. For streams it understands **newline-delimited JSON and whitespace-separated JSON values** — multiple documents in one input are processed in sequence rather than rejected as a parse error[^6].

Because the query layer is a real JavaScript engine, expressions have access to ordinary JS — `.map`, `.filter`, `.reduce`, arrow functions, template literals, arithmetic. There is no jq-style backtracking or path-expression algebra; you compose with JavaScript's own methods instead. The Go binary embeds a JavaScript runtime to evaluate these expressions rather than shelling out to Node.

Distribution is a single static Go binary with no runtime dependency, which is the main practical gain of the rewrite: fast startup, trivial install, and no Node toolchain required for the interactive tool.

## Production Notes

**The npm package is not the interactive tool.** `npm install -g fx` and `deno install -A npm:fx` give you the JavaScript build, which runs in non-interactive mode only[^5]. If you install fx expecting the TUI viewer everyone screenshots and get a plain processor, this is why. For the full tool install the Go binary (Homebrew, `go install`, Scoop, Snap, pacman, Nix, the install script, or Docker)[^7].

**JavaScript numeric semantics apply to expressions.** The viewer is documented to preserve large integers without precision loss, but JavaScript numbers are IEEE 754 doubles. Any value you route through a JS expression is subject to float64 limits, so transforming 64-bit IDs or big integers with a reducer can silently lose precision even though the raw viewer displays them intact. Treat large-integer fields as strings if you need to transform them.

**It is not a drop-in jq replacement for scripting.** jq's DSL is more composable for long, reusable pipelines, is installed nearly everywhere, and has stable, well-specified semantics. fx expressions are JavaScript — great for a quick `.map`/`.filter`, less ideal as portable pipeline code checked into shell scripts. Teams standardizing automation often keep jq for scripts and use fx for interactive exploration.

**Interactive viewing loads the document.** The TUI builds an in-memory tree to fold and search, so opening very large single documents interactively costs memory proportional to the file. Streaming/NDJSON processing is fine for large inputs; interactive exploration of multi-gigabyte files is not the use case.

**Autocompletion is opt-in.** Shell completion for bash, zsh, and fish is generated via `fx --comp <shell>` and must be sourced from your shell rc file[^7]; it is not wired up by the package installers.

## When to Use / When Not

**Use when:**
- You want to interactively explore an unfamiliar JSON/YAML/TOML document — fold, search, and dig without writing a query up front.
- You already know JavaScript and don't want to learn jq's DSL for one-off transforms.
- You're processing NDJSON or streams and want lenient parsing (comments, trailing commas, multiple documents).
- You want a single dependency-free binary rather than a Node install.

**Avoid when:**
- You're writing durable pipeline scripts — jq is more portable, more composable, and more universally installed.
- You need exact big-integer transforms — JS float64 semantics get in the way.
- You only installed the npm build and expected the TUI — that build has no interactive mode.
- You need a read-only viewer with zero eval surface for untrusted input — fx evaluates JavaScript expressions.

## Alternatives

- jqlang/jq — the standard JSON processor; use it when you want a composable, ubiquitous DSL for scripts rather than JavaScript.
- PaulJuliusMartinez/jless — read-only interactive JSON viewer (Rust); use it when you want exploration without any expression-evaluation surface.
- ynqa/jnv — interactive jq REPL (Rust); use it when you want live jq-filter feedback rather than JavaScript.
- tomnomnom/gron — flattens JSON into greppable `path = value` lines; use it when grep/sed pipelines fit better than a query language.
- mikefarah/yq — YAML-first processor with a jq-like syntax; use it when YAML is the primary format and you want in-place edits.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2018-01 | First release as a Node.js JSON processor on npm[^2]. |
| pre-20 | 2018–2022 | JavaScript implementation; interactive mode built on Node. |
| 20.0.0 | 2022 | Complete rewrite in Go; single self-contained binary, TUI viewer as flagship[^4]. |
| 20.x–3x | 2022–2026 | Streaming/NDJSON, YAML & TOML input, comment/trailing-comma tolerance, big-integer display, shell autocompletion[^6]. |

## References

[^1]: fx README and related tools (`walk`, `howto`, `countdown`). https://github.com/antonmedv/fx
[^2]: GitHub API — repository metadata for antonmedv/fx (stars, forks, created 2018-01-25, last push 2026-05). https://github.com/antonmedv/fx
[^3]: fx.wtf — "Write expressions with familiar JavaScript syntax. No need to learn a DSL." https://fx.wtf
[^4]: fx.wtf — "Built using the Go programming language. Distributed as a single self-contained binary." https://fx.wtf
[^5]: fx.wtf/install — "The JavaScript version of fx operates only in a non-interactive mode." https://fx.wtf/install
[^6]: fx.wtf — feature list: streaming/JSON-per-line, YAML and TOML, comments and trailing commas, big-integer precision, long-string handling. https://fx.wtf
[^7]: fx.wtf/install — installation methods (brew, go install, scoop, snap, pacman, nix, docker, npm, deno, install.sh) and `fx --comp` shell completion. https://fx.wtf/install

## Tags

go, cli, json, tui, terminal, json-processor, json-viewer, yaml, command-line, developer-tools, streaming
