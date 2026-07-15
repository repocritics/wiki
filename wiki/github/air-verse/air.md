# air-verse/air

> Live reload for Go apps — a file watcher that rebuilds and restarts your binary on every save.

[GitHub repo](https://github.com/air-verse/air) ·
[License: GPL-3.0](https://github.com/air-verse/air/blob/master/LICENSE)

## Overview

Air is a command-line development tool for Go: run `air` in a project root and it
watches the source tree, re-runs your build command on change, and restarts the
resulting binary. It fills a gap in Go's toolchain — `go run` has no watch mode —
and is most associated with web development on frameworks like Gin, where a manual
stop/rebuild/rerun loop otherwise interrupts every edit. It was written by cosmtrek
(Rui Zhang) around 2017 as a more configurable successor to pilu/fresh[^1].

The scope is deliberately narrow and the README is blunt about it: "This tool has
nothing to do with hot-deploy for production."[^1] Air does not do in-process hot
code swapping (Go has no stable mechanism for that) — it kills the old process and
starts a fresh one. What it adds over a shell loop is debounced watching, per-OS and
per-directory build overrides, `.env` loading, colored streamed logs, and an optional
reverse proxy that injects a browser-reload script.

Two things to know up front. First, Air is GPL-3.0 — unusual in a Go dev-tool
ecosystem that is mostly MIT/BSD. As a standalone CLI you invoke as a separate
process, it does not impose GPL terms on the program you build, but strict tooling
policies should note it. Second, the project is actively maintained (~23.8k stars,
~920 forks, commits landing in mid-2026[^2]) and moved from the personal
`cosmtrek/air` namespace to the `air-verse` organization in 2024; the repo redirects,
the published Docker image is still `cosmtrek/air`, and `go install` targets the new
path.

## Getting Started

```shell
# Recommended: install the binary (requires Go 1.25+)
go install github.com/air-verse/air@latest

# Or as a project-pinned tool dependency (Go 1.25+)
go get -tool github.com/air-verse/air@latest

# Or a prebuilt binary, no Go toolchain needed
curl -sSfL https://raw.githubusercontent.com/air-verse/air/master/install.sh | sh -s -- -b $(go env GOPATH)/bin
```

Make sure `$(go env GOPATH)/bin` is on your `PATH`, then scaffold and run:

```shell
cd /path/to/your_project
air init      # writes .air.toml with defaults
air           # watches, builds, runs; rebuilds on save
```

A minimal `.air.toml` for a typical web server:

```toml
root = "."
tmp_dir = "tmp"

[build]
cmd = "go build -o ./tmp/main ."
entrypoint = ["./tmp/main"]
args_bin = ["server", ":8080"]
exclude_dir = ["assets", "tmp", "vendor", "testdata"]
include_ext = ["go", "tpl", "tmpl", "html"]
delay = 1000            # ms debounce after a change
```

Arguments after `air` are forwarded to the built binary (`air server --port 8080`
runs `./tmp/main server --port 8080`); use `--` to separate Air's own flags from the
app's (`air -c .air.toml -- -h`).

## Architecture / How It Works

Air is a single Go binary with a small, legible core loop:

1. **Watch** — it registers the `root` directory tree (minus `exclude_dir`) with the
   OS filesystem-notification layer (fsnotify) and can pick up directories created
   after startup. Only paths matching `include_ext` / `include_dir` / `include_file`
   trigger a rebuild.
2. **Debounce** — changes within `build.delay` are coalesced, so a multi-file save or
   an editor's atomic-write dance produces one rebuild, not many.
3. **Build** — it shells out to `build.cmd` (any command, not just `go build`), with
   optional `pre_cmd` / `post_cmd` hooks. Compiler stdout/stderr stream to the
   terminal with color; a failed build leaves the previous binary running.
4. **Run** — on success it launches `build.entrypoint` (the modern replacement for the
   deprecated `build.bin`), appending `args_bin` and any passthrough args.
5. **Restart** — on the next qualifying change it signals and kills the running
   process, then repeats from step 3. The build artifact lands in `tmp_dir`.

Configuration is a single TOML file resolved by convention: bare `air` looks for
`.air.toml` in the working directory and falls back to built-in defaults if absent;
`-c` names one explicitly. Every field also has a CLI-flag form
(`air --build.cmd "..." --build.exclude_dir "a,b"`), so the tool is usable with no
config file at all. Platform blocks (`[build.windows]`, `[build.darwin]`,
`[build.linux]`) override base build settings on the matching OS — the reason Windows
users get a separate `.exe` entrypoint.

The optional **proxy** (`[proxy]`) sits in front of your app on a separate port,
injects a reload snippet before `</body>`, and refreshes the browser when watched
static files change — a lightweight LiveReload without a browser extension. This is the
only piece that touches your app's HTTP surface; everything else is pure
process supervision.

## Production Notes

- **Not for production. It is a dev-loop tool.** No process manager, no health checks,
  no supervision guarantees. Use systemd, a container orchestrator, or a real
  supervisor to run built binaries in production.
- **Restart, not hot-swap.** Every change pays a full `go build` plus process
  restart. On large codebases build time dominates the feedback loop; narrowing
  `include_ext`, excluding generated/asset dirs, and tuning `build.delay` matter more
  than anything Air itself does. In-memory state and open connections are lost on each
  restart by design.
- **Watcher limits.** Deep trees can exhaust the OS's inotify watch limit on Linux
  (`fs.inotify.max_user_watches`); a flood of "too many open files" errors usually
  means raising that sysctl or excluding large directories (`node_modules`, `vendor`,
  build output). Editors that save via rename can occasionally desync the watch.
- **Docker/WSL file events are unreliable.** Bind-mounted volumes on Docker Desktop
  and some WSL setups don't reliably propagate native FS events, so saves on the host
  may not reach Air inside the container. The common workaround is running Air on the
  host, or accepting a polling-style setup. The README also notes quoting/escaping
  footguns for binary paths under WSL[^3].
- **`tmp_dir` hygiene.** The build output directory (`tmp/`) should be gitignored and
  excluded from the watch set to avoid rebuild loops; `clean_on_exit` removes it on
  shutdown.
- **Version/config drift.** `go install` requires Go 1.25+. Pinning via `go get -tool`
  keeps a team on one version; installing `@latest` ad hoc invites config-schema
  mismatches. Note `build.bin` is deprecated in favor of `build.entrypoint` and slated
  for removal.

## When to Use / When Not

**Use when:**
- You're iterating on a long-lived Go process (HTTP/gRPC server, worker) and want
  save-triggered rebuild-and-restart.
- You want one tool that also watches templates/static assets and can refresh the
  browser (via the proxy) without extra tooling.
- You need per-OS or per-directory build variations in one declarative config.

**Avoid when:**
- You're shipping to production — this is a development-only loop.
- Your program is a short-lived CLI or one-shot script; a plain `go run` or a shell
  `while` loop is simpler.
- You need genuine hot code reloading with preserved state — Go doesn't support it and
  Air doesn't fake it.
- GPL-3.0 tooling is disallowed by policy and you can't treat it as an external process.

## Alternatives

- githubnemo/CompileDaemon — smaller, single-purpose Go rebuild-on-change; use when you
  want minimal config and no proxy/TOML surface.
- gravityblast/fresh — Air's spiritual predecessor (formerly pilu/fresh); largely
  unmaintained now, choose Air instead unless you already depend on it.
- mitranim/gow — thin `go`-command wrapper (`gow run .`) that adds watching; use when
  you want the `go` CLI's semantics rather than a separate config file.
- watchexec/watchexec — language-agnostic file-watch runner; use when you need to watch
  and re-run arbitrary commands across a polyglot repo, not just Go builds.
- cortesi/modd — general dev-workflow watcher with a concise rules file; use when you're
  orchestrating multiple watch/run rules beyond a single build-and-run.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2017-10 | First release as `cosmtrek/air`, inspired by pilu/fresh[^1]. |
| — | 2024 | Repository moved to the `air-verse` organization; `go install` path changes, Docker image stays `cosmtrek/air`. |
| current | 2026-07 | Actively maintained; `go install` requires Go 1.25+, `build.entrypoint` replaces deprecated `build.bin`[^2]. |

## References

[^1]: air README, "Motivation" and installation/usage sections — air-verse/air. https://github.com/air-verse/air/blob/master/README.md
[^2]: GitHub REST API, repos/air-verse/air (stars, forks, last push, license) — fetched 2026-07-15. https://api.github.com/repos/air-verse/air
[^3]: air README, "Q&A" (WSL escaping, browser auto-reload proxy). https://github.com/air-verse/air/blob/master/README.md

## Tags

go, golang, live-reload, file-watcher, hot-reload, developer-tools, cli, gin, docker, build-tool
