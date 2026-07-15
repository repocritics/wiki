# google/jsonnet

> A purely functional configuration language that extends JSON with variables, functions, and object composition, evaluating to plain JSON.

[GitHub repo](https://github.com/google/jsonnet) ·
[Official website](https://jsonnet.org) ·
[License: Apache-2.0](https://github.com/google/jsonnet/blob/master/LICENSE)

## Overview

Jsonnet is a data templating language created at Google by Dave Cunningham, publicly released in 2014[^1]. It is a strict superset of JSON: every valid JSON document is a valid Jsonnet program. On top of that base it adds variables, arithmetic, conditionals, functions, imports, list/object comprehensions, string formatting, and — its central feature — object composition through inheritance and deep merging. A Jsonnet program is *evaluated* (not "run") to produce a JSON tree, which downstream tools consume as configuration.

The language exists to solve one problem: large configuration corpora (Kubernetes manifests, Grafana dashboards, service configs) accumulate copy-pasted repetition that hand-written JSON/YAML cannot factor out. Jsonnet gives you the abstraction tools of a programming language — while guaranteeing the output is deterministic, side-effect-free JSON. Evaluation is hermetic and lazy: there is no I/O beyond `import`, no clock, no network, so the same inputs always produce the same output.

This repository is the **original C++ implementation** (`libjsonnet`). The maintainers now explicitly recommend the sibling Go reimplementation, google/go-jsonnet, for most users, noting it is "in some cases orders of magnitude faster"[^2]. The C++ tree is still maintained and remains the reference for the C ABI that language bindings link against, but it is no longer where the fastest evaluator lives — a fact worth knowing before you standardize on the `libjsonnet` shared library for a performance-sensitive pipeline.

## Getting Started

```bash
brew install jsonnet          # macOS / Linuxbrew
pip install jsonnet           # Python binding
# or build from source: `make` (GCC) / `make CC=clang CXX=clang++`
```

```jsonnet
// config.jsonnet — a base object extended with the `+` merge operator
local base = {
  replicas: 1,
  image: "nginx:1.25",
  ports: [80],
};

{
  // `base + { ... }` deep-merges; `+:` appends to an inherited field
  dev: base { replicas: 1 },
  prod: base {
    replicas: 5,
    ports+: [443],                      // -> [80, 443]
    image: "nginx:1.25-hardened",
  },
}
```

```bash
jsonnet config.jsonnet          # emits a JSON object with dev/prod keys
jsonnetfmt -i config.jsonnet    # canonical reformat, in place
```

## Architecture / How It Works

Evaluation proceeds in stages: a lexer/parser produces an AST, a desugaring pass rewrites conveniences (comprehensions, `+:`, string interpolation) into a small core language, and a lazy tree-walking interpreter reduces that core to a JSON value. Laziness is load-bearing — object fields are thunks evaluated only when referenced, which is what makes deep inheritance chains and self-references (`self`, `super`, `$`) tractable without infinite loops.

Object composition is the semantic core. `a + b` produces an object where `b`'s fields override `a`'s, `self` late-binds to the final composed object, and `super` reaches the left operand. This is prototype-style inheritance, not templating: a "template" is just a function or object you extend. Fields can be hidden (`::`) so they participate in computation but are stripped from the emitted JSON — the standard way to build up private intermediate values.

The standard library (`std.*`) is the only ambient capability: string manipulation, `std.manifestYamlDoc`, `std.parseJson`, folds, and set operations. There is deliberately no general-purpose I/O. The only escape hatches are `import` (another Jsonnet file), `importstr` (raw text), and `importbin` (bytes) — which read from the filesystem paths the interpreter can reach.

The C++ implementation exposes a stable C ABI (`libjsonnet.h`) that every binding — Python, Go (via cgo historically), Rust, Ruby, PHP, Node — links against. go-jsonnet reimplements the same language spec in pure Go and can also produce a C-compatible `libjsonnet` via cgo, which is how projects transparently swap the faster backend under the same interface.

## Production Notes

**Untrusted input is unsafe.** The README is blunt: the C++ implementation is "not hardened" for evaluating untrusted Jsonnet[^3]. Beyond memory-safety risk, `import`/`importstr`/`importbin` can read any path the process can reach, so a hostile config can exfiltrate secrets. If you accept third-party Jsonnet, sandbox the interpreter (restricted import callback, seccomp, container) — do not rely on the language being "just config."

**Performance cliffs are real.** The C++ evaluator can be pathologically slow on large corpora with heavy object composition — the kind of workload Kubernetes/Grafana config generators produce. This is the primary reason go-jsonnet exists and is recommended. If `jsonnet` is taking seconds-to-minutes on your manifests, switching the binary (or the linked library) to the Go implementation is usually the first and largest win.

**Debuggability is the standing complaint.** Because evaluation is lazy and errors surface at the point of *forcing* a thunk, stack traces can point far from the actual mistake, and a typo in a deeply merged field often manifests as a confusing type error elsewhere. There is no static type system — CUE and Dhall exist partly as answers to this. Teams mitigate with `jsonnetfmt`, disciplined library structure (jsonnet-bundler for versioned deps), and thin per-environment entrypoints.

**Whitespace-in-YAML pitfalls.** Jsonnet emits JSON; when you need YAML (most Kubernetes tooling), you either pipe through a converter or use `std.manifestYamlDoc`, whose formatting quirks (string quoting, multi-doc streams) occasionally bite. Many shops standardize on JSON output and let `kubectl` accept it directly.

**Versioning of the language is slow and stable.** Jsonnet the language changes rarely; most 0.x releases are bug fixes and stdlib additions rather than breaking syntax changes. Upgrade pain is unusual — the more common migration is *implementation* (C++ → Go), not *version*.

## When to Use / When Not

**Use when:**
- You generate large amounts of repetitive JSON/YAML config (Kubernetes, Grafana, Prometheus) and need real abstraction (functions, inheritance, imports).
- You want guaranteed-deterministic, side-effect-free output that diffs cleanly.
- You already live in the Grafana/Tanka or jsonnet-bundler ecosystem.

**Avoid when:**
- You need static types, schema validation, or constraint checking — reach for CUE or Dhall instead.
- You process untrusted config authored by third parties (hardening is on you).
- Your config is small; plain YAML or a thin templating layer (Helm, Kustomize) is less to learn.
- You want readable stack traces and easy debugging for non-experts on your team.

## Alternatives

- google/go-jsonnet — the same language, reimplemented in Go; faster and the maintainers' recommended default. Use this unless you specifically need the C++ tree.
- cue-lang/cue — config language unifying types, values, and constraints; use when you need schema validation and typing that Jsonnet lacks.
- dhall-lang/dhall — total (non-Turing-complete), strongly typed config language; use when you want guaranteed termination and a type checker over raw expressiveness.
- grafana/tanka — Kubernetes config framework built on top of Jsonnet; use when your target is specifically Kubernetes and you want an opinionated workflow.
- kubernetes-sigs/kustomize — overlay-based YAML customization; use when you prefer patching plain manifests over a real language.

## History

| Version | Date | Notes |
|---------|------|-------|
| Initial public release | 2014 | Original C++ implementation open-sourced by Google[^1]. |
| go-jsonnet announced | ~2017 | Pure-Go reimplementation begins; later becomes the recommended evaluator[^2]. |
| 0.16.0 | 2020 | Stdlib growth, `importbin`, formatter maturity (approximate). |
| 0.18–0.20 | 2022–2023 | Continued stdlib additions and bug fixes; language semantics stable. |
| latest 0.21.x | 2024–2026 | Maintenance releases; last repo push 2026-03-30. |

(Exact per-release dates omitted where not verifiable from the fetched metadata; see the repo's Releases page for authoritative tags.)

## References

[^1]: Jsonnet website and history. https://jsonnet.org
[^2]: README recommendation to prefer go-jsonnet ("orders of magnitude faster"). https://github.com/google/go-jsonnet
[^3]: Security notes on untrusted input, from the repository README. https://github.com/google/jsonnet#readme

## Tags

jsonnet, cpp, configuration-language, data-templating, json, functional, kubernetes, config-generation, google, dsl, declarative
