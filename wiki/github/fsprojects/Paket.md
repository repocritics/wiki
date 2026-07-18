# fsprojects/Paket

> A solution-wide dependency manager for .NET that replaced NuGet's silent conflict resolution with a lock file — years before NuGet had one.

[GitHub repo](https://github.com/fsprojects/Paket) ·
[Official website](https://fsprojects.github.io/Paket/) ·
[License: MIT](https://github.com/fsprojects/Paket/blob/master/LICENSE.txt)

## Overview

Paket is a dependency manager for .NET, written in F# and hosted in the fsprojects community incubation space since 2014[^1]. It manages NuGet packages, but also files referenced directly from git repositories, GitHub, and plain HTTP URLs — a capability NuGet still does not have[^2]. Its founding critique of NuGet was specific: `packages.config`-era NuGet tracked dependencies per project, hid which packages were merely transitive, and silently picked a version when two packages disagreed. Paket instead resolves one version per package across the whole solution and pins the full transitive graph in a `paket.lock` file committed to the repo[^3].

The tension defining Paket in 2026 is that NuGet absorbed most of its pitch. PackageReference (2017) made transitive dependencies implicit, `packages.lock.json` (2018) added lock files, and Central Package Management (2022) added solution-wide version control via `Directory.Packages.props`[^4]. What remains genuinely unique is the resolution model (conflicts are errors, not silent upgrades), dependency groups, and git/HTTP file references. Paket today is a deliberate choice — strongly represented in the F# ecosystem (FAKE, many fsprojects repos) — rather than the obvious upgrade it was in 2015.

At ~2.1k stars with the 10.0 release tagged in January 2026[^5], the project is alive but in maintenance-mode cadence: roughly one major version per year since 6.0, mostly runtime retargeting and fixes rather than new capability. The 786 open issues reflect twelve years of accumulated edge cases across every MSBuild era more than active fire.

## Getting Started

Paket is distributed as a .NET tool. The idiomatic setup is a local tool manifest committed to the repo:

```bash
dotnet new tool-manifest        # creates .config/dotnet-tools.json
dotnet tool install paket
dotnet paket init               # creates paket.dependencies
```

`paket.dependencies` (solution root) declares sources and top-level dependencies:

```
source https://api.nuget.org/v3/index.json
storage: none
framework: net8.0

nuget FSharp.Core
nuget Newtonsoft.Json ~> 13.0

// reference a single file straight from GitHub
github fsharp/FAKE src/app/Fake.Core.String/String.fs
```

Each project lists what it consumes in a `paket.references` file next to the project file:

```
FSharp.Core
Newtonsoft.Json
```

Then `dotnet paket install` resolves, writes `paket.lock`, and wires the packages into the projects. `dotnet paket restore` in CI replays the lock file exactly, never re-resolving.

## Architecture / How It Works

Paket is three files plus a resolver:

- **`paket.dependencies`** — the only file you edit. Sources, version constraints (`~>` pessimistic, `>=`, exact), options (`storage`, `framework`), and named **groups** (e.g. a `Build` group for FAKE tooling resolved independently from runtime dependencies).
- **`paket.lock`** — machine-written, committed. The complete resolved graph per group: every package, its version, and its own dependencies. Restore reads only this file, so restores are deterministic and fast[^3].
- **`paket.references`** — per project, names only. Which slice of the solution-wide graph each project consumes.

The resolver solves constraints across the entire group at once. If two dependencies require incompatible versions, resolution fails loudly with the conflict chain — the opposite of NuGet's nearest-wins behavior. The cost: resolution (`paket update`) is the slow path, and on large graphs with wide version ranges it can take minutes; historically the resolver had pathological backtracking cases, substantially improved in the 5.x era.

MSBuild integration happens through `.paket/Paket.Restore.targets`, a committed targets file that hooks the SDK-style restore so `dotnet build` consumes packages from `paket.lock` instead of running NuGet restore[^6]. Before SDK-style projects, Paket rewrote `.csproj`/`.fsproj` reference nodes directly — a design that made Paket 1.x–4.x tightly coupled to project file formats and made the SDK transition a real migration. With `storage: none` packages come from the global NuGet cache; the classic mode extracts one copy of each package into a solution-root `packages/` folder.

## Production Notes

- **One version per group is a policy, not a suggestion.** The whole solution must agree on each package's version. In large monorepos where one project genuinely needs a divergent version, your options are a separate group or a separate solution. Teams either love this discipline or fight it constantly.
- **Ecosystem tooling assumes NuGet.** Dependabot does not support Paket, so automated dependency-update and vulnerability PRs largely don't happen[^7]; `dotnet list package --vulnerable` and similar NuGet-graph tooling don't see Paket's graph. You budget for manual `paket outdated` / `paket update` hygiene.
- **IDE friction.** There is no "Manage NuGet Packages" UI workflow. Adding a package means editing `paket.dependencies`, updating `paket.references`, and running the CLI. Onboarding developers who have never seen Paket is a recurring, real cost on mixed teams.
- **`Paket.Restore.targets` drift.** The targets file is committed and version-coupled to the Paket tool; after upgrading the tool, forgetting to regenerate it produces confusing restore failures. Pin the tool version in `dotnet-tools.json` and regenerate the targets in the same commit.
- **Upgrade pains are era transitions.** 1.x–4.x project-file rewriting → 5.x SDK-style restore targets was the big one; 6.0 (2021) moved distribution to the .NET tool model and made the committed `paket.exe` bootstrapper pattern obsolete. Majors since are mostly runtime retargeting[^5].
- **git/HTTP file dependencies are power and risk.** Referencing source files from arbitrary repos pins to commits in the lock file, but you own availability and provenance of those endpoints — there is no registry guarantee behind them.

## When to Use / When Not

**Use when:**
- You want hard-failure conflict resolution and one authoritative version per package across a solution, enforced by tooling rather than convention.
- You are in the F# ecosystem, where Paket + FAKE is an established, well-trodden stack.
- You need to consume individual files from git/GitHub/HTTP as pinned, lock-file-tracked dependencies.
- You maintain a long-lived Paket codebase — migration off is rarely worth it; the model still works.

**Avoid when:**
- You are starting a mainstream C# project in 2026: NuGet Central Package Management plus `packages.lock.json` covers ~80% of Paket's original value with zero team-onboarding or ecosystem-tooling cost[^4].
- You depend on Dependabot/automated security PRs or NuGet-graph audit tooling.
- Your team churns and nobody wants to own a niche dependency manager.

## Alternatives

- NuGet/Home — the default. With Central Package Management (`Directory.Packages.props`) and `packages.lock.json` it now covers solution-wide versions and deterministic restore; use it unless you need Paket's resolver semantics or git/HTTP references.
- renovatebot/renovate — automated dependency updates for PackageReference-based projects; part of what you give up by choosing Paket.
- dotnet-outdated/dotnet-outdated — CLI for finding stale PackageReference dependencies, the NuGet-side counterpart to `paket outdated`.
- fsprojects/FAKE — not a substitute but the usual companion; Paket groups were designed partly to isolate FAKE build dependencies.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x | 2014-08 | Project started in fsprojects incubation space[^1]. |
| 1.0 | 2015-04 | Stable release; dependencies/lock/references model set[^5]. |
| 2.0 | 2015-09 | `paket pack`/template era consolidation. |
| 3.0 | 2016-06 | .NET Core era begins. |
| 5.0 | 2017-06 | SDK-style project support via `Paket.Restore.targets`; resolver overhaul[^6]. |
| 6.0 | 2021-08 | .NET tool distribution as the primary model. |
| 7.0 | 2022-03 | Runtime retargeting (.NET 6 era). |
| 8.0 | 2023-11 | Runtime retargeting (.NET 8 era). |
| 9.0 | 2024-11 | Maintenance major. |
| 10.0 | 2026-01 | Current major[^5]. |

## References

[^1]: fsprojects — F# Community Project Incubation Space. https://github.com/fsprojects
[^2]: Paket docs, "Git dependencies" / "HTTP dependencies". https://fsprojects.github.io/Paket/git-dependencies.html
[^3]: Paket docs, "The paket.lock file". https://fsprojects.github.io/Paket/lock-file.html
[^4]: NuGet blog, "Introducing Central Package Management" — 2022. https://devblogs.microsoft.com/nuget/introducing-central-package-management/
[^5]: Paket releases (tag dates: 1.0.0 2015-04-17 … 10.0.0 2026-01-19). https://github.com/fsprojects/Paket/releases
[^6]: Paket docs, "Paket and the .NET SDK / dotnet CLI". https://fsprojects.github.io/Paket/paket-and-dotnet-cli.html
[^7]: GitHub docs, "Dependabot supported ecosystems and repositories" (NuGet listed; Paket not supported). https://docs.github.com/en/code-security/dependabot/ecosystems-supported-by-dependabot

## Tags

fsharp, dotnet, package-manager, dependency-management, nuget, lockfile, cli, build-tooling, monorepo
