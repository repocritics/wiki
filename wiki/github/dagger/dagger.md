# dagger/dagger

> A programmable engine that runs your build/test/ship pipelines as typed code inside containers, with content-addressed caching and OpenTelemetry tracing built in.

[GitHub repo](https://github.com/dagger/dagger) ·
[Official website](https://dagger.io) ·
[License: Apache-2.0](https://github.com/dagger/dagger/blob/main/LICENSE)

## Overview

Dagger is an automation engine for software delivery: you write build, test, and deployment logic as functions in a general-purpose language (Go, Python, TypeScript, and others), and the engine executes them as a graph of containerized operations. The pitch is that shell scripts and YAML are a poor substrate for real pipelines, and that a typed, testable programming model with automatic caching is a better one. The same pipeline runs identically on a laptop, in any CI provider, or in the cloud, because the only hard dependency is a Linux container runtime[^1].

The project has an unusually eventful history for a young repo. It was created by Solomon Hykes (a Docker co-founder) and team, and publicly launched in 2022 as a CI/CD engine configured in the CUE data language[^2]. That approach was abandoned within the year: Dagger pivoted to a code-first model where SDKs are generated clients against a GraphQL API served by the engine. Since then it has layered on Functions/Modules (reusable, composable, cross-language units published to a registry called Daggerverse), and more recently added primitives aimed at agentic AI workflows, using the same container-sandbox-plus-cache machinery to run LLM-driven tasks[^3].

The defining tension is scope versus adoption. Dagger asks you to rewrite your CI logic in a real language and adopt its execution model, in exchange for local reproducibility, caching, and portability across CI vendors. That is a large buy-in for teams whose pipelines are a few working YAML files, and the payoff is clearest for organizations drowning in bespoke CI scripts across many repos. As of 2026 the project is still pre-1.0 (0.x versioning), so the API surface and module conventions continue to move.

## Getting Started

```bash
# macOS (Homebrew)
brew install dagger/tap/dagger
# or, any platform:
curl -fsSL https://dl.dagger.io/dagger/install.sh | sh

# Scaffold a module in the language of your choice
dagger init --sdk=go --name=ci
dagger functions        # list the functions the module exposes
```

A generated Go module function. `dag` is the injected client for the engine's API; each call is a node in the pipeline graph, cached by its inputs:

```go
package main

import "context"

type Ci struct{}

// Test runs the project's unit tests inside a pinned container image.
func (m *Ci) Test(ctx context.Context, src *Directory) (string, error) {
	return dag.Container().
		From("golang:1.22").
		WithMountedDirectory("/src", src).
		WithWorkdir("/src").
		WithExec([]string{"go", "test", "./..."}).
		Stdout(ctx)
}
```

```bash
# Call the function; the second run is cached unless inputs changed
dagger call test --src=.
```

## Architecture / How It Works

The Dagger Engine is a long-lived process that runs as a container. It is built on **BuildKit** — the same low-level build engine behind modern `docker build` — which provides the content-addressed cache and the sandboxed execution of each operation[^4]. Dagger wraps BuildKit and exposes a **GraphQL API** describing containers, directories, files, secrets, services, git repositories, and more as typed, composable objects.

SDKs are thin: each is a client generated from the API schema, which is why Dagger can claim SDKs in many languages without maintaining N hand-written implementations. Go, Python, and TypeScript are the mature, first-class SDKs; PHP, Java, .NET, Elixir, and Rust are newer and vary in completeness. When you write a Dagger Function, your code runs *inside a container the engine controls*, and its calls to `dag.*` are GraphQL requests back to the engine over a session — not local library calls.

**Modules and Functions** are the unit of reuse. A module is a package of functions with typed inputs and outputs; modules can call other modules regardless of the language each is written in, because everything crosses the boundary as content-addressed API objects rather than serialized data. Modules are published and discovered through **Daggerverse**. This cross-language composition is Dagger's most distinctive property and the reason the GraphQL indirection exists.

**Caching** is automatic and keyed by inputs. Every operation — pulling an image, mounting a directory, running a command — is content-addressed, so an unchanged input yields a cache hit across local runs and CI without configuration. **Tracing** is built in: each operation emits OpenTelemetry spans, viewable in the CLI's live TUI, in Dagger Cloud, or exported to any OTel backend. The AI/agent additions reuse this exact substrate — an LLM-driven task is just more operations in the same cached, traced, sandboxed graph.

## Production Notes

- **The engine is a container, and it is stateful.** The cache lives in the engine container's volume. On ephemeral CI runners the cache evaporates between jobs unless you persist that volume or point at a shared/remote engine — otherwise you pay full cost every run and lose Dagger's main benefit. Getting caching to actually persist in CI is the most common operational surprise.
- **Docker-in-Docker and privileges.** Because the engine runs containers, CI environments that forbid privileged containers or nested Docker (some hosted runners, restrictive Kubernetes) need deliberate setup. Dagger can run against a remote engine to sidestep this, but that is extra infrastructure.
- **Cold start and image pulls.** First runs pull the engine image and base images before any of your logic executes; on a clean runner this adds noticeable latency that the cache hides only on subsequent runs.
- **Pre-1.0 churn.** The project is still 0.x. Module conventions, SDK APIs, and CLI verbs have changed across minor releases; pinning the Dagger version in CI and reading release notes before upgrading is not optional. The 2022 CUE-to-code pivot is the extreme example, but smaller breaking changes have continued.
- **Debuggability cuts both ways.** OpenTelemetry traces and the TUI make failures far easier to inspect than a wall of CI logs. But when something breaks *inside* the engine or BuildKit layer, you are debugging a container runtime, not a shell script, and the abstraction is deep.
- **It is a programming model, not a config file.** Your pipeline is now code that needs its own dependencies, tests, and review. That is the point, but it means CI logic acquires the maintenance weight of a software project.

## When to Use / When Not

**Use when:**
- You maintain complex pipelines across many repos or CI providers and want one portable, testable implementation instead of divergent YAML.
- Local reproducibility matters — you want `dagger call` on a laptop to behave exactly like CI.
- You value content-addressed caching and end-to-end tracing enough to adopt an execution model to get them.
- You want to compose reusable pipeline modules, possibly across languages, rather than copy-pasting scripts.

**Avoid when:**
- Your CI is a handful of working YAML files; the buy-in outweighs the benefit.
- Your environment forbids privileged/nested containers and you can't stand up a remote engine.
- You need a stable, rarely-changing tool — the 0.x cadence and past pivots imply ongoing churn.
- Your team won't treat pipeline code as real code (tests, review, dependency upkeep); half-adopted, it is just a heavier way to run shell commands.

## Alternatives

- earthly/earthly — Earthfile DSL for repeatable, cached containerized builds; simpler declarative syntax, but not a general-purpose programming model. Use when you want most of the caching/repeatability with a smaller conceptual footprint.
- moby/buildkit — the engine Dagger is built on. Use directly when you only need cached, sandboxed container builds and not cross-language pipeline orchestration.
- bazelbuild/bazel — hermetic, aggressively cached builds across a monorepo. Use when your problem is a fine-grained build graph rather than CI/CD pipelines.
- tektoncd/pipeline — Kubernetes-native pipelines as CRDs. Use when you are all-in on Kubernetes and want CI/CD expressed as cluster resources.
- nektos/act — runs your existing GitHub Actions workflows locally. Use when you only want to reproduce current GH Actions runs on your machine without changing your CI model.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2019-11 | Repository created[^5]. |
| launch | 2022-03 | Public launch as a CI/CD engine configured in the CUE language[^2]. |
| 0.3 | 2022-11 | Pivot to code-first SDKs (Go, Python, TypeScript) over a GraphQL API; CUE-driven approach dropped[^3]. |
| 0.9 | 2023-11 | Functions/Modules introduced as the unit of reusable pipeline logic. |
| 0.10 | 2024-03 | Modules and the Daggerverse module registry reach broad availability. |
| 0.x | 2025 | Primitives for agentic/LLM workflows added atop the existing sandbox-and-cache engine[^3]. |

*Still pre-1.0 as of 2026; no 1.0 release has shipped.*

## References

[^1]: dagger/dagger README — "Runs locally, in your CI server, or directly in the cloud." https://github.com/dagger/dagger
[^2]: Solomon Hykes et al., Dagger public launch, 2022. Company background and CUE-based initial design. https://dagger.io
[^3]: Dagger documentation — SDKs, Functions/Modules, Daggerverse, and the API model. https://docs.dagger.io
[^4]: moby/buildkit — the concurrent, cache-efficient build engine Dagger builds on. https://github.com/moby/buildkit
[^5]: GitHub API `repos/dagger/dagger` — `created_at` 2019-11-20; 16,066 stars, 904 forks, Apache-2.0, Go, last push 2026-07-15 (fetched 2026-07-15).

## Tags

go, ci-cd, devops, containers, buildkit, caching, graphql, pipeline-as-code, automation, developer-tools, agents
