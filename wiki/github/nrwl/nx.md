# nrwl/nx

> A monorepo build system for JavaScript/TypeScript (and polyglot) workspaces: computation caching, an affected-graph, and code generators, with a commercial cloud tier on top.

[GitHub repo](https://github.com/nrwl/nx) ·
[Official website](https://nx.dev) ·
[License: MIT](https://github.com/nrwl/nx/blob/master/LICENSE)

## Overview

Nx is a build orchestrator for monorepos built by Nrwl, a company founded in 2016 by ex-Angular core engineers Victor Savkin and Jeff Cross[^1]. It began as an Angular-first monorepo toolkit and has since become framework-agnostic, with first-party plugins for React, Next.js, Node, Vite, Jest, Playwright, and non-JS toolchains (Gradle, Maven, .NET, Go) discovered through its plugin system. The core value proposition is: build a dependency graph of your projects, hash the inputs of each task, cache the outputs, and only re-run what actually changed.

The defining tension in Nx is between its two operating modes. **Package-based** (or "integrated-lite") usage layers caching and affected-detection onto an existing npm/pnpm/yarn workspace with minimal buy-in — `nx init` reads your existing `package.json` scripts and wraps them. **Integrated** usage adopts Nx's generators, executors, and inferred targets, which delivers the most leverage but also the most coupling: your build config, code scaffolding, and upgrade path all route through Nx abstractions. Teams routinely underestimate how much the integrated style commits them to Nx's conventions.

The second tension is commercial. The Nx CLI is MIT-licensed and fully functional offline, but the highest-value scaling features — remote caching, distributed task execution across machines, and CI-side agents — live in **Nx Cloud**, a paid SaaS product[^2]. The open-source tool is genuinely useful alone; the marketing pushes hard toward the cloud tier. Recent releases lean into an "AI agents" framing (self-healing CI, an MCP server), which is real functionality but also positioning.

## Getting Started

```bash
# add Nx to an existing workspace (package-based, minimal)
npx nx@latest init

# or scaffold a new integrated workspace
npx create-nx-workspace@latest myorg --preset=react-monorepo
```

```bash
# run a target for one project; result is cached on the second run
npx nx build myapp

# run a target only for projects affected by your uncommitted changes
npx nx affected -t test

# visualize the project graph in the browser
npx nx graph
```

A minimal `nx.json` declaring which targets are cacheable and how they depend on upstream projects:

```json
{
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],
      "cache": true,
      "inputs": ["production", "^production"]
    },
    "test": { "cache": true }
  }
}
```

## Architecture / How It Works

Nx's model has a few load-bearing pieces:

- **Project graph.** Nx builds a graph of projects and their dependencies by statically analyzing imports plus explicit config. `affected` commands diff against a base git ref, walk the graph, and compute the minimal set of impacted projects. This graph construction and file hashing were rewritten from TypeScript into **Rust** (the `nx` native addon) around Nx 16 for speed on large repos[^3].
- **Task hashing + cache.** Each task's hash is computed from its declared `inputs` (source files, env vars, dependent projects' outputs, the runtime version, and the command itself). A matching hash restores outputs from cache instead of re-running. The local cache lives on disk; **remote cache** is provided by Nx Cloud (or, at times, self-hostable variants).
- **Executors and generators.** In integrated mode, a project's `build`/`test`/`lint` targets are backed by executors (`@nx/vite:build`, etc.) rather than raw scripts. Generators (`nx g @nx/react:component`) scaffold code against the workspace's conventions.
- **Inferred targets ("Project Crystal").** Introduced in Nx 18 (2024)[^4], plugins infer a project's targets by reading existing tool config (a `vite.config.ts` implies a `build` target) instead of requiring every target to be written out in `project.json`. This reduced config bloat but also made "where did this target come from?" harder to answer — `nx show project myapp --web` is the way to see the resolved config.
- **`nx migrate`.** Automated upgrades: `nx migrate latest` writes a `migrations.json` of codemods that update package versions and rewrite config, which you then run and review. This is one of Nx's genuine differentiators over hand-rolled monorepo tooling.

Nrwl also took over stewardship of **Lerna** in 2022[^5]; Lerna now runs Nx's task runner under the hood, so the two projects are no longer competitors.

## Production Notes

- **Cache correctness depends entirely on declaring inputs correctly.** If a task reads a file or env var that isn't in its `inputs`, Nx will serve a stale cached result. Debugging "it works in CI but not locally" almost always comes back to an unmodeled input. Use `nx reset` to clear the cache when hashes go wrong, and audit `namedInputs` early.
- **Remote cache poisoning is a real risk on shared caches.** A misconfigured or malicious writer can push bad artifacts that other machines then trust. Nx Cloud added cache-integrity controls and access tokens in response; if you self-host a remote cache, treat write access as a security boundary.
- **Version coupling across plugins.** All `@nx/*` plugins are versioned in lockstep with the `nx` core. Mixing versions produces confusing failures. `nx migrate` exists precisely because manual bumps across a dozen plugins are error-prone.
- **Migrations can be large and occasionally lossy.** Major-version migrations rewrite config files; review the diff. Custom executors, patched configs, and non-standard layouts are where migrations break and need manual follow-up.
- **The Nx Cloud upsell is load-bearing for CI scale.** Local caching helps individuals; the payoff on a big team — distributed task execution, agents, cross-run remote cache — requires Nx Cloud. Budget for it or plan CI parallelism yourself. Distributed Task Execution was rebranded to **Nx Agents** in 2024.
- **Graph accuracy vs. dynamic imports.** Static analysis misses dynamically constructed import paths and some non-standard module resolution, which can make `affected` under- or over-select. Explicit `implicitDependencies` are the escape hatch.
- **Windows and native addon.** The Rust native addon ships prebuilt binaries; locked-down or exotic environments (unusual libc, air-gapped installs) occasionally hit binary-resolution issues.

## When to Use / When Not

**Use when:**
- You have many projects/libraries in one repo and rebuild/retest everything on each change.
- Your CI time is dominated by redundant work that caching and affected-detection would cut.
- You want consistent code generation and one-command dependency upgrades across the repo.
- You have a polyglot repo (JS plus Gradle/Maven/.NET/Go) and want one task graph over all of it.

**Avoid when:**
- You have a single app or a handful of packages — pnpm/yarn workspaces plus a thin script are simpler and lock-in-free.
- You want zero vendor gravity; the highest-value CI features pull toward paid Nx Cloud.
- Your team resists framework abstractions and prefers raw, legible build scripts.
- You need hermetic, sandboxed, fully reproducible builds at Google scale — Bazel is the stricter (and harsher) tool for that.

## Alternatives

- vercel/turborepo — lighter, config-first JS/TS monorepo cacher; less generator/plugin machinery, easier to adopt, fewer features. Use when you want caching + affected without buying into an ecosystem.
- bazelbuild/bazel — hermetic, language-agnostic, extremely reproducible; far steeper setup and BUILD-file overhead. Use when correctness/scale demands sandboxed builds.
- lerna/lerna — now Nx-backed; publishing-focused. Use when your main need is versioning/publishing many npm packages.
- moonrepo/moon — Rust-based task runner with a similar affected/cache model. Use when you want Turborepo-like scope with a stricter task schema.
- microsoft/rushjs — pnpm-centric monorepo manager built for very large repos with strict dependency policies. Use when phantom-dependency hygiene matters most.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2018 | First stable release; Angular-focused monorepo toolkit[^1]. |
| 6–8 | 2019 | React support added; broadened beyond Angular. |
| 13 | 2021 | Faster cache, improved task pipeline config. |
| 15 | 2022 | Package-based / standalone (non-monorepo) support; Lerna integration[^5]. |
| 16 | 2023 | Rust-based project graph + hashing native addon[^3]. |
| 18 | 2024 | Project Crystal (inferred targets); Nx Agents (renamed DTE)[^4]. |
| 20 | 2024 | TypeScript project-references workspaces; continuous tasks preview. |
| 21 | 2025 | Continuous tasks, custom version actions, MCP/AI-agent tooling. |

## References

[^1]: Nrwl / Nx origin and founders (Victor Savkin, Jeff Cross, ex-Angular). https://nx.dev/company
[^2]: Nx Cloud — remote caching and CI features. https://nx.dev/nx-cloud
[^3]: Nx blog, "Nx 16" — Rust-based hashing and project graph. https://nx.dev/blog
[^4]: Nx blog, "Project Crystal" (Nx 18, 2024). https://nx.dev/concepts/inferred-tasks
[^5]: "Lerna is dead — long live Lerna" — Nrwl assumes Lerna stewardship, 2022. https://blog.nrwl.io/lerna-is-dead-long-live-lerna-61259f97dbd9

## Tags

typescript, javascript, monorepo, build-system, build-tool, caching, ci, task-runner, code-generator, developer-tooling
