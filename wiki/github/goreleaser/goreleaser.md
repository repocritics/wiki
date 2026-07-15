# goreleaser/goreleaser

> Declarative release automation for Go (and increasingly non-Go) projects — one YAML file turns a git tag into cross-compiled binaries, archives, packages, containers, and a published release.

[GitHub repo](https://github.com/goreleaser/goreleaser) ·
[Official website](https://goreleaser.com) ·
[License: MIT](https://github.com/goreleaser/goreleaser/blob/main/LICENSE.md)

## Overview

GoReleaser is a build-and-release orchestrator created by Carlos Alexandro Becker (caarlos0) in late 2016[^1]. Its premise: the mechanical parts of shipping a release — cross-compiling for every OS/arch, tarring up binaries with a license and README, generating checksums, signing them, building Docker images, cutting a GitHub/GitLab/Gitea release, and bumping a Homebrew tap — are all deterministic and should be described once in a `.goreleaser.yaml` and executed by a single `goreleaser release` on tag push. It became the de facto release tool for Go CLIs and is bundled into thousands of CI pipelines via the official GitHub Action[^2].

Historically GoReleaser was Go-only, and the Go toolchain's trivial cross-compilation (`GOOS`/`GOARCH`) is what made it possible. Since the 2.x line it has generalized: custom and prebuilt builders let it package Rust, Zig, Python, and TypeScript artifacts too[^3], though the ergonomics are still best for Go. The tool's real value is not any single step but the composition — the same config produces reproducible, signed, multi-platform artifacts locally (`--snapshot`) and in CI without change.

The defining tension is the open-core model. The core tool is MIT-licensed and complete for most single-repo projects, but a paid tier — GoReleaser Pro — gates a set of features aimed at larger or more complex setups. Deciding a release pipeline around GoReleaser means accepting that some scaling paths lead to a paywall.

## Getting Started

```bash
# install (macOS/Linux via Homebrew; see site for apt/yum/scoop/go install)
brew install goreleaser/tap/goreleaser

# scaffold a .goreleaser.yaml in your repo
goreleaser init

# dry-run: build everything locally, skip publishing, allow a dirty tree
goreleaser release --snapshot --clean
```

```yaml
# .goreleaser.yaml
version: 2

builds:
  - env: [CGO_ENABLED=0]
    goos: [linux, darwin, windows]
    goarch: [amd64, arm64]
    ldflags:
      - -s -w -X main.version={{.Version}} -X main.commit={{.Commit}}

archives:
  - formats: [tar.gz]
    format_overrides:
      - goos: windows
        formats: [zip]

checksum:
  name_template: "checksums.txt"

release:
  github:
    owner: myorg
    name: mytool
```

A real release is then `git tag -a v1.2.3 -m ... && git push origin v1.2.3` followed by `goreleaser release --clean` (usually run by CI, not a human).

## Architecture / How It Works

GoReleaser is a linear pipeline of independent "pipes." A run executes them in a fixed order, each consuming the artifacts the previous pipes registered: load config → apply defaults → resolve git state (last tag, commit, dirty check) → `before` hooks → build → universal-binary merge → archive → nfpm packages (deb/rpm/apk) → checksum → sign → SBOM → Docker build/push → SBOM/sign for images → publish (release upload, Homebrew/Scoop/AUR/nix, Snapcraft, blobs) → announce (Twitter/X, Slack, Discord, Telegram, etc.)[^4]. Pipes that aren't configured are skipped; there is no plugin runtime — every capability is compiled in.

Configuration is Go `text/template`-driven. Fields like `name_template`, `ldflags`, and tag names are evaluated against a context carrying `.Version`, `.Tag`, `.Commit`, `.Date`, `.Os`, `.Arch`, and environment variables. This templating is powerful and is also the primary source of confusing errors, since a typo in a template surfaces late in the run.

Version discovery is git-tag-based and opinionated: GoReleaser reads the current tag, refuses to publish from a dirty working tree, and expects semver-ish tags. The `--snapshot` flag bypasses the tag/clean requirements for local testing and CI dry-runs; `--skip=publish,sign,...` selectively disables pipes. `goreleaser build` runs only the build pipe (useful as a cross-compiler), and `goreleaser check` validates the config against the current schema — important because the config format is versioned (`version: 2`) and deprecations are enforced.

## Production Notes

- **Deprecations are real churn.** GoReleaser removes deprecated config keys on a schedule, and a config that worked a year ago can fail `check` after a major bump. `version: 2` (the 2.x line) was itself a breaking cleanup that dropped long-deprecated fields[^5]. Pin the GoReleaser version in CI (the action's `version:` input) rather than tracking `latest`, or a release can break on an unrelated day.
- **Open-core paywall on scaling paths.** Features including split-and-merge builds (parallelizing a single release across multiple CI runners and recombining), monorepo support (multiple configs / tag prefixes in one repo), nightly builds, and `includes` (importing shared config fragments) are GoReleaser Pro, not the OSS tool[^6]. Teams often discover this only when they hit the exact scaling problem Pro solves. Budget for it or design around it early.
- **CGO and cross-compilation.** The happy path assumes `CGO_ENABLED=0` pure-Go builds. The moment you need CGO you must supply cross-compilers/toolchains yourself (zig cc, musl, osxcross), and per-target builds multiply CI time. This is a Go-toolchain constraint GoReleaser inherits, not one it solves.
- **Token scope and publishing.** `release` uploads require a token with contents write; Homebrew tap / Scoop bucket updates require a token that can push to a *different* repo. Cross-repo publishing failures (403s late in the run, after artifacts are built) are a common first-release footgun. Use a PAT or a scoped app token, not the default `GITHUB_TOKEN`, for tap updates.
- **Docker builds are sequential and host-arch bound by default.** Multi-arch images need buildx/QEMU set up in the environment; GoReleaser wires the manifests but does not provision the emulation. Image builds run in the same process, so a slow registry push serializes the tail of your release.
- **Reproducibility caveats.** `-trimpath` and `-X ...{{.Date}}` are common, but embedding `{{.Date}}` defeats reproducible builds; use `{{.CommitDate}}` and `SOURCE_DATE_EPOCH` if bit-for-bit reproducibility matters.

## When to Use / When Not

**Use when:**
- You ship a Go CLI or service and want tagged, cross-platform, checksummed, optionally signed releases from CI with one config.
- You want Homebrew/Scoop/AUR/nfpm distribution without hand-writing each formula/manifest.
- You want a local `--snapshot` that produces the exact artifacts CI will, for testing.

**Avoid when:**
- Your project isn't Go and doesn't fit the prebuilt/custom builder model — a language-native tool (cargo-dist for Rust, electron-builder, etc.) will be smoother.
- You need monorepo, split-machine, or nightly builds and won't pay for Pro.
- Your release logic is heavily bespoke/imperative — a plain Makefile or a `dagger`/`task` pipeline may fit better than GoReleaser's declarative, fixed-order model.

## Alternatives

- axodotdev/cargo-dist — Rust-native equivalent; use when your project is Cargo-based and you want installers/shell-script installers out of the box.
- electron-userland/electron-builder — use for desktop Electron apps needing code-signing and auto-update, which GoReleaser doesn't target.
- go-task/task or plain Make — use when you want full imperative control over release steps and don't need the packaging/publishing breadth.
- semantic-release/semantic-release — use when your primary need is automated version bumping and changelog from commits (npm-centric); pair with, rather than replace, a builder.
- jreleaser/jreleaser — JVM-world analog with a similar declarative release model; use in Java/Kotlin/native-image projects.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.1.0 | 2016-12-29 | Initial release; Go binaries + GitHub release[^1]. |
| v1.0.0 | 2021-11-14 | First stable line; config stabilization and deprecation policy. |
| v2.0.0 | 2024-06-05 | Breaking cleanup; dropped long-deprecated config, `version: 2` schema, broader non-Go builder support[^5]. |
| v2.16.0 | 2026-05-24 | 2.x feature line (see release notes). |
| v2.17.0 | 2026-07-04 | Latest stable at time of writing. |

GoReleaser Pro is developed alongside the OSS tool and shares the same binary, activating extra pipes when a Pro key is present.

## References

[^1]: goreleaser/goreleaser release history — v0.1.0 published 2016-12-29. https://github.com/goreleaser/goreleaser/releases
[^2]: GoReleaser GitHub Action. https://github.com/goreleaser/goreleaser-action
[^3]: GoReleaser docs, "Builders" (Go, custom, and prebuilt for other languages). https://goreleaser.com/customization/builds/
[^4]: GoReleaser docs, customization index (pipeline stages: builds, archives, nfpms, checksum, sign, sbom, dockers, release, announce). https://goreleaser.com/customization/
[^5]: GoReleaser blog, "GoReleaser v2" — 2024-06. https://goreleaser.com/blog/goreleaser-v2/
[^6]: GoReleaser Pro features and pricing. https://goreleaser.com/pro/

## Tags

go, release-automation, ci-cd, cross-compilation, cli-tooling, packaging, devops, build-tool, open-core, github-actions, docker, homebrew
