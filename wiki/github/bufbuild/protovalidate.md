# bufbuild/protovalidate

> Runtime validation for Protobuf messages, driven by CEL expressions instead of generated code.

[GitHub repo](https://github.com/bufbuild/protovalidate) ·
[Official website](https://protovalidate.com) ·
[License: Apache-2.0](https://github.com/bufbuild/protovalidate/blob/main/LICENSE)

## Overview

Protovalidate is Buf's validation system for Protocol Buffers, and the declared successor to protoc-gen-validate (PGV)[^1]. You annotate `.proto` fields and messages with validation rules — either standard rules (`string.email`, `uint32.lte`, `repeated.min_items`) or custom rules written in Google's Common Expression Language (CEL)[^2] — and a runtime library evaluates them against each message. It targets anyone who already speaks Protobuf/gRPC/Connect and wants a single, language-agnostic source of truth for "what makes this message valid" that lives in the schema rather than scattered across hand-written service code.

The defining architectural bet is *runtime evaluation over code generation*. PGV was a `protoc` plugin: it emitted a `Validate()` method in each target language at build time, which meant every language needed its own generator and the generated code could drift. Protovalidate ships the validation constraints as Protobuf annotations (the `buf/validate/validate.proto` schema) and interprets them at runtime via a CEL engine plus reflection over the message descriptor. No generated validation code, one shared rule schema, and — critically — a conformance test suite that every language runtime must pass so behavior is identical across Go, Java, Python, C++, and TypeScript/JavaScript[^3].

This repository is the hub, not a runtime library. It holds the `buf/validate` proto definitions (the standard rules and their semantics), the CEL standard-library extensions, the conformance harness, and the documentation. The actual libraries you import live in separate per-language repos: protovalidate-go, protovalidate-java, protovalidate-python, protovalidate-cc, and protovalidate-es. Star counts and issue activity here reflect the schema/spec project, not any single runtime.

## Getting Started

The schema is published on the Buf Schema Registry as `buf.build/bufbuild/protovalidate`; add it as a dependency and annotate your messages:

```protobuf
syntax = "proto3";
package acme.user.v1;

import "buf/validate/validate.proto";

message User {
  string id    = 1 [(buf.validate.field).string.uuid = true];
  uint32 age   = 2 [(buf.validate.field).uint32.lte = 150];
  string email = 3 [(buf.validate.field).string.email = true];

  option (buf.validate.message).cel = {
    id: "first_name_requires_last_name"
    message: "last_name must be present if first_name is present"
    expression: "!has(this.first_name) || has(this.last_name)"
  };
}
```

Then validate at runtime — Go shown here (each language ships its own package):

```go
import "buf.build/go/protovalidate"

v, err := protovalidate.New()
if err != nil { /* handle */ }
if err := v.Validate(user); err != nil {
    // err is a *protovalidate.ValidationError with per-field violations
    log.Printf("invalid user: %v", err)
}
```

## Architecture / How It Works

The pipeline in every runtime is roughly: read the `buf.validate` extensions off the message/field descriptors, compile each standard rule and each custom `cel` expression into a CEL program once, cache those compiled programs keyed by message type, then execute them against each incoming message. `this` binds to the field or message under evaluation. Standard rules are themselves implemented in terms of CEL under the hood — the "standard library" is a set of predefined expressions plus a few custom CEL functions (for example `isEmail`, `isHostname`, `isIp`) that would be awkward or slow to express in pure CEL.

Because rules are compiled from descriptors at runtime, the runtime needs the `buf.validate` extension definitions available in the descriptor set. In Go this means the generated `.pb.go` for your messages must be linked with the protovalidate extension types; in dynamically-loaded scenarios (e.g. validating `dynamicpb` messages from a `FileDescriptorSet`) you must ensure the validate extensions are registered in the resolver, which is a common first-run stumbling block.

The conformance suite is the load-bearing part of the project. It defines a corpus of messages and expected validation outcomes and drives each language runtime as a subprocess, so "the Go and Python behavior differ" is treated as a bug in a runtime rather than an accepted quirk. This is what lets teams mix languages across a gRPC mesh and trust that a message rejected by one service would be rejected by all of them.

## Production Notes

- **First-call compilation cost.** CEL programs are compiled lazily on first validation of a given message type. High-percentile latency on a cold path can spike. Most runtimes let you warm the cache — in Go, pass `WithMessages(...)` to `New()` to precompile known types at startup rather than paying it on the first request.
- **Reuse the validator.** The validator object holds the compiled-program cache. Constructing a new one per request throws that cache away and recompiles every time; build it once and share it (it is safe for concurrent use in the mature runtimes).
- **Runtime versions move independently of this repo.** The schema repo hit v1.0.0 in September 2025, but each language library has its own version line and its own v1.0 timeline. Pin the runtime, and check that the runtime version you use supports the schema features (e.g. newer standard rules) you annotate with — a message using a rule the runtime predates will not be enforced as expected.
- **Migrating from protoc-gen-validate is not a drop-in.** The annotation namespace changed (`validate.rules` → `buf.validate.field`), some rule names and semantics differ, and PGV's generated `Validate()` calls disappear in favor of a runtime validator object. Buf ships a migration guide and tooling, but treat it as a real migration with test coverage, not a find-and-replace[^4].
- **CEL is powerful enough to be a footgun.** Custom expressions run on every message; an expensive expression (large `repeated` traversals, regex-heavy checks) becomes per-request cost. Keep custom rules cheap, and prefer standard rules where they exist since those are optimized.
- **Not a substitute for authorization or business rules that need I/O.** CEL has no network/database access by design. Validation here is structural/semantic on the message itself; anything requiring external lookups belongs in application code.

## When to Use / When Not

**Use when:**
- You have a polyglot gRPC/Connect surface and want one schema-level definition of validity enforced identically everywhere.
- You are starting new Protobuf APIs, or already on Buf tooling (BSR, `buf lint`, `buf breaking`).
- You want validation rules to travel with the schema (versioned, reviewable, generated into docs) rather than living in service code.
- You are migrating off protoc-gen-validate, which is now in maintenance.

**Avoid when:**
- You are not using Protobuf — this is Protobuf-specific and offers nothing for JSON Schema / OpenAPI stacks.
- Your validation needs external state (DB uniqueness, cross-service checks); CEL cannot do I/O.
- You need one specific niche language with no protovalidate runtime yet; the five supported languages are the boundary.
- Ultra-tight latency budgets where you cannot tolerate any interpretation overhead and would hand-write checks.

## Alternatives

- bufbuild/protoc-gen-validate — the predecessor; code-gen instead of runtime CEL. Use only for legacy projects already committed to it; new work should prefer protovalidate.
- envoyproxy/protoc-gen-validate — the same PGV lineage under Envoy's org; same "use if already invested" caveat.
- go-playground/validator — struct-tag validation for Go. Use when your source of truth is Go structs, not Protobuf schemas.
- google/cel-go — the underlying CEL engine. Use directly when you need general-purpose expression evaluation beyond message validation.
- protocolbuffers/protobuf — vanilla Protobuf gives you type/required-field checks only; use alone when you have no semantic-rule needs.

## History

| Version | Date | Notes |
|---------|------|-------|
| announced | 2023-10 | Introduced as the CEL-based successor to protoc-gen-validate[^1]. |
| v0.1.0 | 2023-11 | Early schema/conformance releases; runtimes in beta. |
| v0.10.0 | 2025-02 | Late-0.x line; rule set and conformance suite maturing. |
| v1.0.0-rc.1 | 2025-06-04 | First release candidate for the stable schema. |
| v1.0.0 | 2025-09-12 | Stable 1.0 of the schema and conformance suite. |
| v1.1.0 | 2025-12-09 | Post-1.0 additions. |
| v1.2.0 | 2026-04-15 | Continued rule/tooling additions. |
| v1.2.2 | 2026-07-09 | Latest patch at time of writing. |

## References

[^1]: Buf, "Protovalidate: parameterized, runtime Protobuf validation" — announcement, 2023. https://buf.build/blog/protovalidate-v1
[^2]: Common Expression Language project. https://cel.dev/
[^3]: Protovalidate documentation and language overview. https://protovalidate.com/
[^4]: Buf, "Migrate from protoc-gen-validate" guide. https://protovalidate.com/migration-guides/migrate-from-protoc-gen-validate/

## Tags

go, java, python, cpp, typescript, protobuf, cel, validation, grpc, schema, code-generation, buf
