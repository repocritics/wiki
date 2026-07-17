# bufbuild/buf

> A Protobuf toolchain — compiler, linter, breaking-change detector, formatter, and code generator — that replaces day-to-day `protoc` scripting.

[GitHub repo](https://github.com/bufbuild/buf) ·
[Official website](https://buf.build) ·
[License: Apache-2.0](https://github.com/bufbuild/buf/blob/main/LICENSE)

## Overview

Buf is a Go command-line tool for working with Protocol Buffers, built by Buf (the company) around a reimplementation of the Protobuf compiler[^1]. It targets the gap between "I have `.proto` files" and "I have governed, versioned APIs": instead of hand-maintained `protoc -I ...` shell scripts, you declare a workspace in `buf.yaml` and run `buf build`, `buf lint`, `buf breaking`, `buf format`, and `buf generate` against it. The schema language and the generated-code plugin model are unchanged; Buf sits above them as workflow tooling.

The defining tension is open-core. The CLI is Apache-2.0 and fully functional offline — compile, lint, format, breaking-change detection, and local code generation need no account. But the features Buf markets most heavily — remote plugins, hosted generated SDKs, dependency resolution for private modules, hosted docs, and server-side schema checks — live in the **Buf Schema Registry (BSR)**, a commercial hosted product with a free tier[^2]. This is the same "great on the vendor's platform, do-it-yourself elsewhere" dynamic that surrounds many open-core tools: the local CLI is genuinely useful on its own, but the recommended happy path pulls you toward buf.build.

At ~11.3k stars and near-daily commits as of mid-2026, Buf is the de facto modern alternative to raw `protoc`, and the reference implementation for ConnectRPC and Protobuf-ES, which the same company maintains.

## Getting Started

```sh
brew install bufbuild/buf/buf   # or: npm i -g @bufbuild/buf, or download a release binary
```

```sh
buf config init                              # writes a buf.yaml
buf build                                    # compile the workspace
buf lint                                     # 40+ built-in rules
buf format -w                                # rewrite files in place
buf breaking --against '.git#branch=main'    # compare against a git ref
buf generate                                 # run plugins from buf.gen.yaml
```

A minimal `buf.yaml` (v2 config format):

```yaml
version: v2
modules:
  - path: proto
lint:
  use: [STANDARD]
breaking:
  use: [FILE]
```

## Architecture / How It Works

Buf does **not** shell out to `protoc`. It ships its own Protobuf compiler (the `protocompile` library) written in Go, built for deterministic, parallel compilation and tested against `protoc`'s descriptor output[^1]. The unit of work is a *module* (a tree of `.proto` files with a `buf.yaml`), and a project is a *workspace* of one or more modules. A single config drives build, lint, breaking, generation, and publishing so they all agree on the same input set and import resolution — eliminating the classic `-I` path drift where import order silently changes behavior.

Code generation preserves the standard plugin protocol: `buf generate` constructs a `CodeGeneratorRequest` and feeds it to plugins exactly as `protoc` would, so any conforming `protoc-gen-*` binary works. What changes is that plugins, outputs, options, and inputs move into a checked-in `buf.gen.yaml` instead of a long command line. Plugins can be **local** (installed binaries) or **remote** (`remote: buf.build/protocolbuffers/go`), where the remote form runs generation on Buf's servers so no generator binaries need to be installed on developer machines or CI runners.

Breaking-change detection is the most distinctive piece. Protobuf compatibility is not one property: renaming a field breaks generated *source* while keeping the *wire* format intact; changing `int32` to `string` breaks every serialized message. `buf breaking` splits this into rule categories — `FILE`, `PACKAGE`, `WIRE_JSON`, and `WIRE` — and `--against` accepts a git ref, a BSR module, a tarball, a zip, a directory, or a prebuilt image, so the same command runs on a laptop, in CI, and in release automation.

The BSR is a Protobuf-aware registry: it stores modules, verifies they compile, renders docs, resolves `buf.lock` dependencies, hosts remote plugins, produces per-language generated SDKs consumable from `go get` / `npm install` / Maven / pip / Cargo, and can enforce checks server-side before a breaking change reaches consumers.

## Production Notes

**Open-core coupling is the main operational decision.** Everything local works without an account, but if you adopt remote plugins or hosted generated SDKs, your builds now depend on buf.build availability and your continued relationship with a vendor. Teams that want the ergonomics without the coupling pin **local** plugins in `buf.gen.yaml`. Self-hosting the BSR exists only as an enterprise offering — there is no open-source registry server in this repo.

**`buf breaking` needs real git history.** `--against '.git#branch=main'` fails or gives wrong answers on the shallow clones many CI systems default to. Set `fetch-depth: 0` (or equivalent) or the check silently compares against an incomplete tree.

**The `STANDARD` lint ruleset is opinionated.** It enforces package version suffixes, directory-matches-package, PascalCase messages, and more. Pointing it at a mature `.proto` tree typically surfaces a large backlog on day one; teams usually start from a smaller rule set (or `buf lint` ignores) and ratchet up, rather than turning STANDARD on everywhere at once.

**Managed mode is powerful and a footgun.** It lets API producers strip language-specific file options (`go_package`, `java_package`, …) out of `.proto` files and inject them centrally in `buf.gen.yaml`. When consumers also set those options, or when overrides are misconfigured, generated package names come out wrong in ways that are hard to trace back to the config.

**Config v1 → v2 migration.** The `buf.yaml` / `buf.gen.yaml` v2 format (module lists in one file, revised workspace semantics) is a genuine migration; `buf migrate` automates most of it, but multi-module repos should budget review time. Older tutorials and copy-pasted configs are frequently v1.

**Stability promise.** Since v1.0, Buf commits to no breaking CLI changes until a v2.0 it says it has no plans to ship[^3]. The exception is anything behind the `buf beta` gate, where interfaces change without notice — do not build load-bearing automation on `buf beta` commands.

## When to Use / When Not

**Use when:**
- You maintain shared `.proto` APIs and want lint + breaking-change gates in CI.
- You want to delete bespoke `protoc` shell scripts in favor of one declarative config.
- You want Connect / gRPC clients and servers generated reproducibly across languages.
- You're willing to adopt the BSR (or deliberately stay local-only).

**Avoid when:**
- You have a handful of `.proto` files and a working `protoc` line — the added toolchain may not pay off.
- You need a bleeding-edge `protoc` feature or a plugin that assumes the real `protoc`; Buf's compiler is a faithful reimplementation but not byte-identical in every exotic case.
- You require a fully self-hostable, no-vendor registry — the BSR is hosted/commercial.
- Your build already resolves proto through Bazel rules and you don't want a second source of truth.

## Alternatives

- protocolbuffers/protobuf — the canonical `protoc` compiler and runtimes; use directly when you want no extra layer or need the newest upstream compiler behavior.
- grpc/grpc — use when you're committed to gRPC's own multi-language codegen and tooling rather than Connect/Buf ergonomics.
- uber/prototool — Buf's spiritual predecessor (lint/breaking/format for proto); deprecated in favor of Buf, historical only.
- bazelbuild/rules_proto — use when your build already lives in Bazel and you want proto compilation as build rules instead of a standalone CLI.
- connectrpc/connect-go — not a replacement but a companion; Buf's RPC framework that consumes the schemas Buf manages.

## History

| Version | Date | Notes |
|---------|------|-------|
| repo created | 2019-10-03 | Initial `buf` development (lint + breaking around own compiler)[^4]. |
| v1.0.0 | 2022 | First stable CLI; begins the no-breaking-changes-until-v2 promise[^3]. |
| v1.x (BSR era) | 2022–2024 | Remote plugins, generated SDKs, hosted docs, dependency management mature on the BSR. |
| buf.yaml v2 | 2024 | New config format consolidating multi-module workspaces; `buf migrate` added. |
| ongoing | 2026 | Actively maintained; ~11.3k stars, near-daily commits[^4]. |

## References

[^1]: Buf compiler / `protocompile` — Buf's Go reimplementation of the Protobuf compiler. https://github.com/bufbuild/protocompile
[^2]: Buf Schema Registry documentation. https://buf.build/docs/bsr/
[^3]: Buf README, "CLI stability" — no breaking changes within a major version; no planned v2.0. https://github.com/bufbuild/buf#cli-stability
[^4]: bufbuild/buf repository metadata (stars, creation date, last push) via GitHub API, retrieved 2026-07-17. https://github.com/bufbuild/buf

## Tags

go, protobuf, grpc, protoc, code-generation, linter, schema-registry, api-tooling, cli, connectrpc, breaking-change-detection
