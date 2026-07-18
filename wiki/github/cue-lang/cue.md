# cue-lang/cue

> A configuration language where types are values and merging is unification — schemas, constraints, and data in one lattice, with no inheritance to debug.

[GitHub repo](https://github.com/cue-lang/cue) ·
[Official website](https://cuelang.org) ·
[License: Apache-2.0](https://github.com/cue-lang/cue/blob/master/LICENSE)

## Overview

CUE ("Configure, Unify, Execute") is a data validation and configuration language created by Marcel van Lohuizen, who previously worked on GCL, Google's internal Borg configuration language[^1]. CUE is explicitly a reaction to GCL's failure mode: inheritance and overrides make large configs unanalyzable because a value's final state depends on evaluation order. CUE's answer is to make configuration a **value lattice**: every CUE expression is a value, types are just less-specific values (`int` is a value that subsumes `3`), and combining configs is unification — commutative, associative, and idempotent, so composition order never changes the result[^2]. There is deliberately no inheritance and no user-defined functions.

The project started under the `cuelang` GitHub org in 2018 and moved to `cue-lang/cue` in July 2021. It is written in Go, doubles as a Go library (`cuelang.org/go`), and interoperates with JSON, YAML, TOML, JSON Schema, OpenAPI, Protobuf, and Go types — `cue import` converts these into CUE, and `cue export` emits them back out. Its natural habitat is Kubernetes-scale YAML sprawl: define a schema once, validate thousands of manifests, and generate them with defaults filled in.

At 6.2k stars it is smaller than its mindshare suggests — CUE is plumbing that ships inside other tools (Timoni, KubeVela, Grafana's schema work, early Dagger) more often than it is adopted directly. Development is active and commercially backed by CUE Labs: v0.17.1 shipped 2026-07-16 and the repo sees near-daily pushes, but the language is still pre-1.0 after eight years, and minor releases carry semantic changes.

## Getting Started

```bash
brew install cue           # or: go install cuelang.org/go/cmd/cue@latest
```

```cue
// config.cue
#Deployment: {
	name:     string
	replicas: int & >=1 & <=10 | *3   // constraint with default 3
	env:      "dev" | "staging" | "prod"
}

api: #Deployment & {
	name: "api"
	env:  "prod"
}
```

```bash
cue vet config.cue            # validate — fails if any value violates a constraint
cue export config.cue         # {"api": {"name": "api", "replicas": 3, "env": "prod"}}
cue vet schema.cue data.yaml  # validate existing YAML against a CUE schema
```

## Architecture / How It Works

**Everything is a value.** CUE's core is a lattice ordered by specificity: `_` (top, any value) ⊒ `string` ⊒ `"hello"` ⊒ `_|_` (bottom, conflict). Unification (`&`) computes the greatest lower bound of two values; disjunction (`|`) offers alternatives; `*x` marks a default within a disjunction. Validation is not a separate phase — a config is valid iff its unification with the schema is not bottom. This design descends from typed feature structures in computational linguistics[^2].

**Definitions and closedness.** `#Foo` declares a definition; structs unified with a definition are *closed* — unknown fields are errors, which is how CUE catches typos in manifests. `...` reopens a struct. `field!: type` marks a field as required (v0.6.0, 2023)[^3].

**Two evaluators.** The original evaluator (evalv2) had superlinear blowups on comprehension- and disjunction-heavy configs — the main historical complaint against CUE at scale. A rewrite (evalv3) with structure sharing and better disjunction handling was introduced behind `CUE_EXPERIMENT=evalv3` in v0.9.0 and became the default in v0.13.0 (2025-05), fixing dozens of bugs and improving performance for most workloads[^4]. `CUE_EXPERIMENT=evalv3=0` reverts.

**Modules.** CUE modules are distributed via OCI registries, with the Central Registry (registry.cue.works, run by CUE Labs) as the default[^5]. Experimental support landed in v0.8.0; the experiment was enabled by default in v0.9.0 (2024-06)[^6]. `cue mod init` / `cue mod tidy` mirror the Go modules workflow.

**Tooling layer.** `_tool.cue` files plus `cue cmd` define scriptable workflows (fetch, deploy, diff) inside CUE itself — a small, undertested corner of the project compared to the evaluator. The Go API (`cue.Context`, `cue.Value`, `Unify`, `Decode`) is how most embedding tools consume CUE.

## Production Notes

- **Pre-1.0 semantics churn.** Minor releases have changed closedness rules, `let` scoping, and evaluator behavior. Pin the `cue` binary version in CI and treat upgrades as migrations; the evalv2→evalv3 switch has a dedicated upgrade FAQ because edge-case results differ[^4].
- **Performance cliffs.** evalv3 removed the worst blowups, but large disjunctions (e.g. a field accepting one of hundreds of variants) and deep comprehensions can still be slow. Profile with realistic config sizes before committing to CUE as a render path in a hot loop.
- **Error messages.** Conflict errors report the full unification chain ("conflicting values X and Y"), which is precise but hard to read in thousand-line configs. Expect a learning-curve tax for on-call engineers who did not write the schemas.
- **Closedness footgun.** Definitions close structs recursively; teams routinely get "field not allowed" errors when layering configs and reach for `...` everywhere, which silently discards the typo protection that motivated definitions in the first place.
- **No functions.** Reuse happens through definitions, comprehensions, and the builtins. Genuinely algorithmic transformations must move into Go via the API — by design, but it caps how much logic a pure-CUE codebase can express.
- **Ecosystem signal is mixed.** Dagger, the highest-profile early adopter, moved its primary user interface from CUE to language SDKs in 2022. Timoni (Kubernetes package manager) and KubeVela remain committed CUE consumers. Air-gapped module use requires self-hosting an OCI registry instead of the Central Registry.

## When to Use / When Not

**Use when:**
- You validate or generate large volumes of JSON/YAML (Kubernetes, CI pipelines, service configs) and want schema + data in one language.
- You need order-independent composition — many teams layering config without override wars.
- You are embedding config validation in a Go program; the Go API is first-class.
- You want to converge JSON Schema / OpenAPI / Protobuf definitions into one source of truth via `cue import`.

**Avoid when:**
- You need functions, loops with state, or arbitrary computation in config — Jsonnet, Nickel, or a real language serve better.
- Your configs are small enough that plain YAML plus a JSON Schema validator covers the need; CUE's concepts are a real onboarding cost.
- You require a stability guarantee; pre-1.0 semantic changes and the evaluator migration are ongoing operational facts.
- Non-engineers edit the config; the lattice model and error output assume developer readers.

## Alternatives

- google/jsonnet — use instead when you want template-style config generation with functions and inheritance-like overrides.
- dhall-lang/dhall-haskell — use instead when you want a total, strongly typed functional config language with hermetic imports.
- tweag/nickel — use instead when you want CUE-like contracts plus real user-defined functions.
- kcl-lang/kcl — use instead when you want a CNCF-hosted, Rust-based config language with a more imperative feel for Kubernetes.
- hashicorp/hcl — use instead when the config is consumed by Terraform-family tooling rather than needing standalone validation.

## History

| Version | Date | Notes |
|---------|------|-------|
| pre-0.4 | 2018–2021 | Developed under the `cuelang` org; language design iterations. |
| 0.4.0 | 2021-07 | First release after the move to the `cue-lang` org. |
| 0.6.0 | 2023-08 | Required fields (`field!: type`)[^3]. |
| 0.8.0 | 2024-03 | Experimental OCI-based modules[^6]. |
| 0.9.0 | 2024-06 | Modules on by default; evalv3 introduced as experiment. |
| 0.13.0 | 2025-05 | evalv3 evaluator becomes the default[^4]. |
| 0.17.0 | 2026-06 | Current minor line (v0.17.1: 2026-07-16). |

## References

[^1]: CUE documentation, "The Logic of CUE". https://cuelang.org/docs/concept/the-logic-of-cue/
[^2]: CUE Language Specification. https://cuelang.org/docs/reference/spec/
[^3]: CUE v0.6.0 release notes — required fields. https://github.com/cue-lang/cue/releases/tag/v0.6.0
[^4]: CUE v0.13.0 release notes — "enables the new evaluator by default"; upgrade FAQ: https://cuelang.org/docs/concept/faq/upgrading-from-evalv2-to-evalv3/ · https://github.com/cue-lang/cue/releases/tag/v0.13.0
[^5]: CUE modules reference. https://cuelang.org/docs/reference/modules/
[^6]: CUE v0.8.0 and v0.9.0 release notes — modules experiment and default-on. https://github.com/cue-lang/cue/releases/tag/v0.9.0

## Tags

go, configuration-language, data-validation, schema, kubernetes, json, yaml, unification, devops, cli
