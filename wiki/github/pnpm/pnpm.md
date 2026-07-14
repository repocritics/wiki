# pnpm/pnpm

> A Node.js package manager that trades npm's flat `node_modules` for a symlinked, content-addressable store — saving disk space and catching phantom dependencies, at the cost of tools that assume a hoisted layout.

[GitHub repo](https://github.com/pnpm/pnpm) ·
[Official website](https://pnpm.io) ·
[License: MIT](https://github.com/pnpm/pnpm/blob/main/LICENSE)

## Overview

pnpm ("performant npm") is a drop-in alternative to the npm CLI, created by Zoltán Kochan and first released in 2016[^1]. It installs the same packages from the same npm registry and reads the same `package.json`, but differs in two structural decisions: packages are stored once in a global content-addressable store and linked into projects, and `node_modules` is a nested symlink tree rather than a flat, hoisted directory. Together these give the two headline properties — disk efficiency (one copy of each package version per machine) and strictness (code can only import dependencies it actually declared).

The defining tension is that strictness. npm and Yarn Classic hoist all transitive dependencies to the top level of `node_modules`, so a package can `require()` something it never listed in `package.json` as long as some other package pulled it in — a "phantom dependency." pnpm's default isolated layout makes only declared dependencies resolvable, which surfaces real correctness bugs but also breaks packages and tools that relied (often unknowingly) on hoisting. Most pnpm friction, historically, traces back to this one design choice.

pnpm's other center of gravity is monorepos. Workspaces, a shared lockfile, filtered commands (`pnpm --filter`), and catalogs make it a common choice for large multi-package repos; Microsoft's Rush uses it under the hood[^2]. GitHub reports the repository's primary language as Rust, which reflects `pacquet`, an experimental Rust port of the CLI vendored in-tree — the shipping pnpm itself is TypeScript.

## Getting Started

```bash
# Standalone install (no Node required), or via Corepack / npm
curl -fsSL https://get.pnpm.io/install.sh | sh -
# or: corepack enable pnpm
# or: npm install -g pnpm
```

```bash
pnpm install                 # install from package.json + pnpm-lock.yaml
pnpm add react               # add a dependency
pnpm add -D typescript       # dev dependency
pnpm dlx create-vite my-app  # npx equivalent (fetch + run, no install)
pnpm run build               # run a script
```

A minimal monorepo is declared in `pnpm-workspace.yaml`:

```yaml
# pnpm-workspace.yaml
packages:
  - "packages/*"
  - "apps/*"
catalog:
  react: ^19.0.0   # one version, referenced as "catalog:" in each package.json
```

```bash
pnpm -r build                     # run "build" in every workspace package
pnpm --filter @acme/web dev       # run only in one package (and its deps)
```

## Architecture / How It Works

**Content-addressable store.** Every file of every package version is stored once under a global store (default `~/.local/share/pnpm/store` on Linux, configurable via `store-dir`), keyed by a hash of its contents. Installing links files from the store into `node_modules` using hard links or, on copy-on-write filesystems, reflinks. Two projects depending on the same lodash version share the same on-disk inode; a new version that changes one file adds only that one file to the store[^3]. The hard-link mechanism requires the store and the project to live on the **same filesystem** — if they differ (e.g. store on the system drive, project on an external volume), pnpm silently falls back to copying, losing the space savings.

**The virtual store and symlink layout.** `node_modules/.pnpm` is a flat "virtual store" holding every package version in a directory like `.pnpm/lodash@4.17.21/node_modules/lodash`. Each package's own dependencies are symlinked next to it inside `.pnpm`, and the project's direct dependencies are symlinked from `node_modules/` into `.pnpm`. Node's default module resolution follows the symlinks, so a package resolves exactly the dependency versions it declared — no more, no less. This is `node-linker=isolated`, the default.

**Alternative linkers.** `node-linker=hoisted` reproduces npm's flat layout for tools that cannot cope with symlinks; `node-linker=pnp` uses Yarn's Plug'n'Play resolution. `public-hoist-pattern` and the blunt `shamefully-hoist=true` selectively raise packages to the top level for compatibility.

**Lockfile.** `pnpm-lock.yaml` pins the full dependency graph. Its format is versioned and has changed with major releases (lockfile v6 in pnpm 8, v9 in pnpm 9), so committing a lockfile written by a newer pnpm and installing with an older one triggers a rewrite.

**Peer dependencies.** pnpm resolves peers per-consumer: if two packages depend on the same library but with different peer contexts, pnpm may create multiple instances of a package in the store to keep peer resolution correct, which is more precise than npm's dedup but can surprise code that assumes a singleton.

## Production Notes

**Symlinks break some tooling.** The isolated layout is the top source of incidents. Bundlers and runtimes that don't preserve symlinks, resolve realpaths inconsistently, or assume a flat tree can misbehave — React Native's Metro, some Jest module setups, certain Docker `COPY node_modules` flows, and tools that walk `node_modules` naively. The escape hatches are `node-linker=hoisted` or targeted `public-hoist-pattern`, at the cost of reintroducing phantom-dependency risk.

**Phantom dependencies surface on migration.** Moving an existing npm/Yarn project to pnpm frequently produces "module not found" errors for packages that always worked — because they were phantom. The fix is to add the missing packages to `package.json` (correct), not to reach for `shamefully-hoist` (a workaround that masks the bug).

**Lifecycle scripts are not run by default (pnpm 10+).** Starting with pnpm 10, dependencies' `postinstall`/build scripts do not execute unless the package is allowlisted via `onlyBuiltDependencies` or approved with `pnpm approve-builds`[^4]. This is a supply-chain hardening measure, but it silently breaks packages that compile native binaries or download assets on install (esbuild, some native addons) until you approve them — a common first-run surprise after upgrading.

**Pin the pnpm version in CI.** Because lockfile format and default behaviors shift across majors, CI must use the same pnpm version as developers. Use the `packageManager` field in `package.json` with Corepack, or pin explicitly in the CI setup step; a mismatch causes lockfile churn or `--frozen-lockfile` failures.

**`--frozen-lockfile` in CI.** pnpm enables it automatically when `CI` is set: installs fail rather than mutate the lockfile if `package.json` and `pnpm-lock.yaml` disagree. This is the intended behavior but trips teams that forgot to commit a lockfile update.

**Store maintenance.** The global store grows unbounded; `pnpm store prune` removes unreferenced files. On CI, caching the store (not `node_modules`) plus `--frozen-lockfile` gives the fastest reliable installs.

## When to Use / When Not

**Use when:**
- You run a monorepo — workspaces, filtering, catalogs, and a single lockfile are first-class.
- Disk space or install speed across many projects/CI runs matters (shared store, hard links).
- You want strictness that catches undeclared dependencies before they reach production.
- You want a mostly npm-compatible CLI without adopting a fundamentally different model (as Yarn Berry PnP or Bun ask you to).

**Avoid when:**
- Your toolchain assumes a hoisted/flat `node_modules` and can't be configured (some React Native, legacy bundlers) — you'll spend the savings fighting symlinks.
- You need the single fastest installer above all and can adopt its ecosystem — Bun's installer is faster in many benchmarks.
- The team is small, the project is a single package, and npm's defaults already work — the migration cost may not pay back.

## Alternatives

- npm/cli — the default; flat hoisted layout, no shared store, weaker monorepo story. Use when zero migration cost matters more than strictness or disk usage.
- yarnpkg/berry — Yarn 2+/PnP; zero-install and strict resolution, but a larger departure from Node's default resolution. Use when you want PnP or Yarn's plugin system.
- oven-sh/bun — Bun's built-in installer is very fast and npm-compatible. Use when raw install speed dominates and you're comfortable in Bun's ecosystem.
- microsoft/rushstack (Rush) — orchestration layer for very large monorepos that runs on top of pnpm. Use when you outgrow plain workspaces and need build orchestration/publishing policy.
- vercel/turborepo — task runner/caching for monorepos; complements rather than replaces pnpm. Use alongside pnpm when you need remote build caching.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2017 | First stable release; symlinked `node_modules`[^1]. |
| 4.0 | 2019 | Content-addressable store refinements, faster installs. |
| 5.0 | 2020 | Non-flat isolated layout hardened; workspaces maturing[^3]. |
| 7.0 | 2022 | Node.js 14+ baseline, config and lockfile updates. |
| 8.0 | 2023 | Lockfile v6, settings cleanup. |
| 9.0 | 2024-05 | Lockfile v9, catalogs (9.5), default store/layout changes. |
| 10.0 | 2025-01 | Dependency lifecycle scripts off by default (approve-builds)[^4]. |
| 11.x | 2026 | Runtime/Node version manager surface (`pnpm runtime`)[^5]. |

## References

[^1]: Zoltán Kochan and contributors, pnpm project history. https://pnpm.io/motivation
[^2]: Rush — "Why pnpm?" / Rush uses pnpm in large monorepos. https://rushjs.io/pages/maintainer/package_managers/
[^3]: pnpm blog, "Flat node_modules is not the only way" — 2020-05-27. https://pnpm.io/blog/2020/05/27/flat-node-modules-is-not-the-only-way
[^4]: pnpm docs, settings — `onlyBuiltDependencies` / `pnpm approve-builds`. https://pnpm.io/settings#onlybuiltdependencies
[^5]: pnpm docs (11.x), "pnpm runtime". https://pnpm.io/11.x/cli/runtime

## Tags

javascript, typescript, package-manager, nodejs, monorepo, dependency-management, content-addressable-storage, cli, npm, workspaces
