# lerna/lerna

> The original JavaScript monorepo tool — now a versioning-and-publishing layer
> on top of the Nx task runner.

[GitHub repo](https://github.com/lerna/lerna) ·
[Official website](https://lerna.js.org) ·
[License: MIT](https://github.com/lerna/lerna/blob/main/LICENSE)

## Overview

Lerna manages multiple JavaScript/TypeScript packages in a single repository:
running scripts across packages in dependency order, computing version bumps,
generating changelogs, and publishing to npm. It emerged from the Babel
project's own multi-package tooling in late 2015 and for roughly five years was
the default answer to "how do we do a JS monorepo," predating native
workspaces in npm, Yarn, and pnpm.

The project's defining story is a near-death and revival. Maintenance stalled
around 2020, and in May 2022 stewardship transferred to Nrwl, the company
behind Nx[^1]. The new maintainers wired Lerna's task execution into the Nx
runner (opt-in in 5.1[^2], default in 6.0) and then, in the v7 release, deleted
Lerna's own package-management layer entirely — `lerna bootstrap`, `lerna add`,
and `lerna link` are gone, replaced by package-manager workspaces[^3]. Modern
Lerna is therefore a much smaller tool than its reputation suggests: `version`
and `publish` are its differentiated core, while `run` is a thin veneer over Nx.

That coupling cuts both ways. The Nx takeover is why Lerna is alive — releases
are steady (v9.0.7 shipped 2026-03[^6], pushes land daily as of mid-2026) — but
installing Lerna now means installing Nx, and much of its 36k-star reputation
was earned by a pre-2020 tool that no longer exists in the same form.

## Getting Started

```bash
npx lerna init          # scaffolds lerna.json + package.json workspaces

npx lerna run build                      # run "build" in each package, topologically, cached
npx lerna run test --scope=@myorg/core   # filter to one package
npx lerna version --conventional-commits # bump versions + changelogs from commit history
npx lerna publish from-package           # publish whatever isn't on the registry yet
```

Versioning mode lives in `lerna.json`: `"version": "independent"` or a fixed
number like `"1.2.3"`. Packages are discovered through your package manager's
`workspaces` configuration (`package.json` or `pnpm-workspace.yaml`) — Lerna
no longer installs or links dependencies itself[^3].

## Architecture / How It Works

Lerna is a command suite over three subsystems:

- **Package graph.** Lerna reads the workspace globs, builds a dependency graph
  from `package.json` files, and uses it everywhere: topological ordering for
  `run`, change detection for `version`, publish ordering for `publish`.
- **Task running via Nx.** `lerna run` delegates to the Nx task runner, which
  provides parallelism, task pipelines (`dependsOn`), and local computation
  caching keyed on file inputs. Delegation became the default in v6 and the
  legacy runner was subsequently removed; an `nx.json` file unlocks advanced
  configuration (cacheable operations, named inputs, remote caching via Nx
  Cloud). Lerna and Nx share the same graph machinery — `nx` is a direct
  dependency of the `lerna` package.
- **Versioning and publishing.** Two modes: **fixed** (one version number for
  the whole repo, stored in `lerna.json` — the Babel model) and
  **independent** (each package versions separately). `lerna version` detects
  changed packages since the last git tag, computes bumps (interactively or
  from Conventional Commits), rewrites `package.json` files, generates
  changelogs, commits, and tags[^4]. `lerna publish` pushes to npm in
  dependency order, with `from-git` / `from-package` recovery modes for CI.

The v7 amputation of `bootstrap`/`add`/`link` is the architectural watershed:
pre-7 Lerna duplicated package-manager work (hoisting, symlinking) with its own
semantics; post-7 Lerna trusts workspaces and does only what package managers
don't — orchestrated versioning and publishing[^3].

## Production Notes

- **The v6 → v7 upgrade is the big one.** If your CI runs `lerna bootstrap`,
  it breaks. Migration means adopting real workspaces (and absorbing your
  package manager's hoisting semantics, which differ from Lerna's legacy
  hoisting). `@lerna/legacy-package-management` exists as a stopgap, but it is
  frozen — treat it as a migration ramp, not a destination[^3].
- **Cache correctness is your problem.** Nx caching defaults are conservative,
  but once you declare `cacheableOperations` you must also declare task
  `outputs` and inputs accurately in `nx.json`; undeclared outputs silently
  don't restore on cache hits, which surfaces as "works on rebuild, fails on
  cache hit" CI mysteries.
- **Publishing under npm 2FA.** Interactive OTP prompts don't work in CI; use
  granular automation tokens or trusted publishing/provenance. After a partial
  publish failure (one package published, five didn't), `lerna publish
  from-package` is the recovery path — publishing is not transactional.
- **Fixed vs independent has long-term costs.** Fixed mode bumps every package
  on any release, producing version churn in packages that didn't change.
  Independent mode with `--conventional-commits` produces bump cascades through
  internal dependents and noisier changelogs. Teams switch modes rarely and
  painfully; choose deliberately.
- **Nx surface area.** Debugging `lerna run` means debugging Nx: daemon
  processes, `.nx/cache` state (`nx reset` clears it), and version
  compatibility between the bundled `nx` and any Nx plugins you add.
- **Maintenance reality check.** 288 open issues and a same-week push cadence
  (as of 2026-07) indicate active but Nx-prioritized maintenance;
  Lerna-specific features move slower than Nx-core ones.

## When to Use / When Not

**Use when:**

- You publish many interdependent npm packages and want automated,
  graph-ordered `version` + `publish` with Conventional Commits changelogs.
- You have an existing Lerna repo on v7/v8 — staying current is cheaper than
  migrating off.
- You want task caching without hand-assembling Nx, and don't mind Nx being
  underneath.

**Avoid when:**

- You don't publish to a registry — an app-only monorepo gets the same task
  running from Nx or Turborepo directly, without the publishing machinery.
- You want PR-driven, human-curated release notes — changesets' file-per-change
  model fits review workflows better than commit-message inference.
- You want minimal dependencies — pnpm workspaces plus `pnpm -r run` covers
  small monorepos with no extra tool.
- You are on Lerna ≤6 expecting `bootstrap` to keep working — that tool is
  gone; budget a migration either way.

## Alternatives

- changesets/changesets — use instead when you want intent files reviewed in
  PRs to drive versioning, rather than Conventional Commits inference.
- nrwl/nx — use instead when you want the full task-graph system directly,
  with plugins and generators, and publishing is secondary (`nx release` now
  covers it).
- vercel/turborepo — use instead when you only need fast cached task running
  in an app monorepo and no npm publishing.
- pnpm/pnpm — use instead when workspace linking plus recursive script
  execution is enough and you want zero additional tooling.
- microsoft/rushstack — use instead for large enterprise monorepos wanting
  strict install/version policies (Rush) over convention.

## History

| Version | Date | Notes |
|---------|------|-------|
| 2.0 | 2017-07 | First stable major after a long beta; `bootstrap`-centric era. |
| 3.0 | 2018-08 | Modular rewrite (`@lerna/*` packages), `lerna version` split from `publish`. |
| 4.0 | 2021-02 | Maintenance release during the low-activity period. |
| 5.0 | 2022-05 | First release under Nx (Nrwl) stewardship[^1]. |
| 5.1 | 2022-06 | Optional Nx task runner (`useNx`), new website[^2]. |
| 6.0 | 2022-10 | Nx runner by default; caching and task pipelines mainstream. |
| 7.0 | 2023-06 | Removes `bootstrap`/`add`/`link`; workspaces required[^3]. |
| 8.0 | 2023-11-23 | Alignment with Nx 17-era internals[^5]. |
| 9.0 | 2025-09-23 | Current major line; 9.0.7 latest as of 2026-03[^6]. |

## References

[^1]: "Lerna is looking for new stewardship" → resolved: Nrwl takes over — lerna/lerna#3121, 2022-05. https://github.com/lerna/lerna/issues/3121
[^2]: Nrwl blog, "Lerna 5.1 — New website, new guides, distributed caching support" — 2022-06. https://blog.nrwl.io/lerna-5-1-new-website-new-guides-new-lerna-example-repo-distributed-caching-support-and-speed-64d66410bec7
[^3]: Lerna docs, "Legacy Package Management" — rationale for removing `bootstrap`/`add`/`link` in v7. https://lerna.js.org/docs/legacy-package-management
[^4]: Lerna docs, "Version and Publish". https://lerna.js.org/docs/features/version-and-publish
[^5]: Lerna v8.0.0 release — 2023-11-23. https://github.com/lerna/lerna/releases/tag/v8.0.0
[^6]: Lerna v9.0.0 (2025-09-23) and v9.0.7 (2026-03-13) releases. https://github.com/lerna/lerna/releases/tag/v9.0.0

## Tags

typescript, javascript, monorepo, build-system, package-publishing, versioning, npm, task-runner, developer-tools, cli
