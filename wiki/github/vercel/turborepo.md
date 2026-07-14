# vercel/turborepo

> A task runner and incremental build system for JavaScript/TypeScript monorepos, built around content-hash caching.

[GitHub repo](https://github.com/vercel/turborepo) ·
[Official website](https://turborepo.dev) ·
[License: MIT](https://github.com/vercel/turborepo/blob/main/LICENSE)

## Overview

Turborepo is a build orchestrator for JavaScript and TypeScript monorepos. It does not compile, bundle, or transpile anything itself — it wraps the scripts you already have (`build`, `test`, `lint`) and decides which of them can be skipped, run in parallel, or restored from cache. The core idea is that most tasks in a monorepo are pure functions of their inputs, so their outputs can be hashed and cached: change one package, and only that package and its dependents need to re-run[^1].

It was created by Jared Palmer as an independent project and acquired by Vercel in December 2021[^2]. The original implementation was written in Go; Vercel rewrote the engine in Rust over 2023–2024, which is why the repo's primary language now shows as Rust despite the tool being distributed as an npm package (`turbo`)[^3]. The user-facing behavior stayed stable across that rewrite.

The defining tension is scope. Turborepo is deliberately thin: it schedules and caches tasks but has no opinion about your bundler, test runner, package manager, or code generation. This makes it easy to adopt on an existing repo but means it does not solve dependency graph management, code generation, or affected-detection at the depth that heavier tools (Nx, Bazel) do. You bring the tools; Turborepo makes them not re-run.

## Getting Started

```bash
# add to an existing pnpm/npm/yarn workspace
pnpm add turbo --save-dev --workspace-root
# or scaffold a new monorepo
npx create-turbo@latest
```

```jsonc
// turbo.json — task graph definition (Turborepo 2.x schema)
{
  "$schema": "https://turborepo.dev/schema.json",
  "tasks": {
    "build": {
      "dependsOn": ["^build"],       // run deps' build first (topological)
      "outputs": ["dist/**", ".next/**"]
    },
    "test": {
      "dependsOn": ["build"],
      "outputs": []                   // no outputs → cache result/logs only
    },
    "dev": { "cache": false, "persistent": true }
  }
}
```

```bash
turbo run build test --filter=@acme/web...   # web + everything it depends on
```

## Architecture / How It Works

Turborepo builds a task graph from two inputs: the workspace's package dependency graph (read from `package.json` workspaces plus the lockfile) and the `tasks` definitions in `turbo.json`. The `dependsOn` field distinguishes topological dependencies (`^build` = "the `build` task of my dependencies") from same-package ordering (`build` = "my own build task first").

Caching is the whole point. For each task, Turborepo computes a hash over the package's source files, its resolved dependencies, the task's environment variables, the `turbo.json` config, and the hashes of upstream tasks it depends on. If that hash has been seen before, the declared `outputs` (plus captured stdout/stderr) are restored from the cache and the script is not executed — this is a full cache hit, reported as `>>> FULL TURBO`. Because the hash includes dependency hashes, a change deep in the graph correctly invalidates everything downstream.

Caches are content-addressable and stored locally in `node_modules/.cache/turbo` by default. The same artifacts can be pushed to a **Remote Cache** so CI and teammates share hits. Vercel hosts a Remote Cache as a managed service, but the protocol is documented and self-hostable — several open-source cache servers implement it[^4].

The Rust core handles graph construction, hashing, file globbing, and process scheduling. Environment variables are a first-class part of the hash: by default Turborepo warns about env vars used by a task but not declared, because an undeclared var that affects output is a correctness hole (you can get a stale cache hit). Declaring `env` / `globalEnv` keys is how you make that explicit.

## Production Notes

**Cache correctness depends on honest `outputs` and `env` declarations.** The most common failure is a task whose real output isn't fully listed in `outputs` (so a restored cache hit is missing files) or a task that reads an undeclared environment variable (so it gets a hit it shouldn't). Turborepo cannot see inside your build tool; it only knows what you declare. Treat cache-related bugs as declaration bugs first.

**Remote cache poisoning across environments.** If local dev and CI differ in ways not captured by the hash (different OS, different tool versions installed globally, differing `NODE_ENV`), a machine can pull a cache entry that doesn't match its environment. Include the significant env vars in `env`, and be cautious about sharing a single remote cache between fundamentally different platforms.

**The 1.x → 2.0 config break.** Turborepo 2.0 (2024) renamed the top-level `pipeline` key to `tasks` and tightened several defaults[^5]. Repos upgrading from 1.x need the `turbo` codemod or a manual edit; a 1.x `turbo.json` will not run on 2.x unchanged. This is the upgrade most teams hit.

**It does not replace your package manager.** Turborepo relies on pnpm/npm/yarn workspaces for installation and hoisting; it reads the lockfile but does not manage `node_modules`. Workspace misconfiguration (phantom dependencies, wrong lockfile) surfaces as Turborepo problems but must be fixed at the package-manager layer.

**`--filter` is powerful and easy to get wrong.** The filter syntax (`...`, `^`, scope globs, changed-since git ranges) controls which tasks run. Overly broad filters defeat the incremental model; overly narrow ones skip work that should run. It rewards learning the syntax precisely.

**Watch mode and persistent tasks.** Long-running `dev` servers must be marked `persistent: true` and `cache: false`; otherwise Turborepo will try to treat them as cacheable finite tasks.

## When to Use / When Not

**Use when:**
- You have an existing JS/TS monorepo (pnpm/yarn/npm workspaces) and re-running unchanged tasks in CI is the bottleneck.
- You want caching and parallelism without rewriting how packages build.
- You want remote cache sharing between CI and developers with minimal setup.

**Avoid when:**
- Your repo is a single package — there is no graph to exploit; the overhead isn't worth it.
- You need polyglot builds (Go, Java, Python) with fine-grained affected detection — Bazel or Nx fit better.
- You want built-in code generation, dependency graph visualization, and executors — Nx ships those; Turborepo intentionally doesn't.

## Alternatives

- nrwl/nx — heavier, batteries-included monorepo toolkit with generators, executors, and affected graphs; choose it when you want opinionated tooling rather than a thin task runner.
- microsoft/rush — enterprise-scale npm monorepo manager with strict dependency policy; choose it for very large repos needing phantom-dependency guarantees.
- moonrepo/moon — Rust task runner with a language-agnostic bent; choose it when you want Turborepo-style caching beyond just JS/TS.
- bazelbuild/bazel — hermetic, polyglot, correctness-first build system; choose it when reproducibility and multi-language builds matter more than setup cost.
- pnpm/pnpm — its workspace `--filter` and `-r` can run tasks across packages; choose it when you don't need cross-run caching at all.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2021-12 | Vercel acquires Turborepo; first release under Vercel[^2]. |
| Go→Rust | 2023–2024 | Engine incrementally rewritten in Rust[^3]. |
| 2.0 | 2024-06 | `pipeline` renamed to `tasks`; config and default changes[^5]. |

## References

[^1]: Turborepo docs, "Caching" — how task hashing and cache restoration work. https://turborepo.dev/docs/crafting-your-repository/caching
[^2]: Vercel blog, "Vercel acquires Turborepo" — 2021-12-09. https://vercel.com/blog/vercel-acquires-turborepo
[^3]: Turborepo docs / Vercel engineering on the Go-to-Rust migration of the core. https://turborepo.dev/blog
[^4]: Turborepo docs, "Remote Caching" — the cache protocol and self-hosting. https://turborepo.dev/docs/core-concepts/remote-caching
[^5]: Turborepo blog, "Turborepo 2.0" — 2024-06-04, including the `pipeline`→`tasks` rename. https://turborepo.dev/blog/turbo-2-0

## Tags

rust, javascript, typescript, monorepo, build-system, task-runner, caching, ci, vercel, developer-tools
