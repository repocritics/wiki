# jesseduffield/lazydocker

> A terminal UI for Docker and Docker Compose — a read-mostly cockpit that turns a handful of `docker` and `docker compose` incantations into one keypress each.

[GitHub repo](https://github.com/jesseduffield/lazydocker) ·
[License: MIT](https://github.com/jesseduffield/lazydocker/blob/master/LICENSE)

## Overview

lazydocker is a single-binary terminal UI (TUI) for managing Docker containers, images, volumes, networks, and Docker Compose projects. It is written in Go by Jesse Duffield — the same author as lazygit — and shares that project's design language: a keyboard-driven panel layout, a config-driven custom-command system, and mouse support as an afterthought that happens to work[^1]. The pitch, taken from the README's own "minor rant," is that keeping track of containers across multiple terminal windows and half-remembered `docker-compose` flags is a chore, and that a single always-on dashboard removes most of it[^2].

The tool sits deliberately in a narrow band. It is not an orchestrator, not a replacement for `docker` in scripts, and not a Kubernetes tool. It is a human-facing operator console for a single Docker daemon — usually your local one during development. Everything it does is achievable with raw CLI commands; the value proposition is entirely about reducing keystrokes and context-switching, and about surfacing state (logs, stats, exit codes) that you would otherwise have to poll for manually.

The defining tension is scope discipline versus feature creep. After roughly seven years the project remains pre-1.0 (latest tag v0.25.2 as of April 2026)[^3], which the author treats as a signal of "small, stable, done enough" rather than immaturity. With ~52k stars it is one of the most-starred Docker developer tools that is not made by Docker itself, yet it is maintained at a low-intensity, occasional-release cadence — a fact worth weighing if you need responsive upstream support.

## Getting Started

```sh
# macOS / Linux (Homebrew)
brew install lazydocker

# Windows
scoop install lazydocker        # or: choco install lazydocker

# From source (Go >= 1.19)
go install github.com/jesseduffield/lazydocker@latest
```

Then run it in any directory:

```sh
lazydocker
```

If a `docker-compose.yml` sits in the working directory, lazydocker picks it up automatically and shows a per-service view; otherwise it shows all containers on the daemon. A common ergonomic touch is aliasing it:

```sh
echo "alias lzd='lazydocker'" >> ~/.zshrc
```

## Architecture / How It Works

lazydocker is a client of the Docker Engine API, not a wrapper around the `docker` CLI for most operations. It talks to the daemon over `/var/run/docker.sock` (or `DOCKER_HOST`) using the official Go SDK, and renders panels with gocui — the terminal-layout library maintained in the lazy* ecosystem[^1]. The UI is a fixed set of side panels (Project, Containers, Images, Volumes, Networks) plus a main view that reflows to show logs, stats, config, or the top process list for the selected item.

Three implementation choices drive most of its behavior:

- **Polling, not events.** State is refreshed by repeatedly querying the daemon on a timer rather than subscribing to the Docker event stream for everything. This keeps the code simple and the UI predictable, but it means very short-lived containers or rapid restarts can be missed between refreshes, and a busy daemon adds steady background API traffic.
- **Log and stat streaming.** For the selected container, logs and CPU/memory stats are streamed continuously and drawn as scrolling text and ASCII graphs. Switching selection tears down and re-establishes those streams.
- **Custom commands as config.** Nearly every action — restart, remove, exec a shell, prune — is expressed as a templated shell command in `config.yml`, not hardcoded. Users can add their own `customCommands` per context (container, service, image), which is the main extensibility surface[^4]. Compose operations in particular are shell-outs to `docker compose`, so lazydocker inherits whatever Compose version is on your PATH.

Configuration lives at `~/.config/jesseduffield/lazydocker/config.yml` (XDG-respecting). There is no plugin system and no daemon of its own; the binary is the entire program. This monolithic, config-over-code shape is why the tool is easy to install and hard to break, and also why deep customization means learning its Go-template command syntax rather than writing extensions.

## Production Notes

lazydocker is a developer-workstation tool first, and its rough edges appear when people push it past that.

- **It manages one daemon at a time.** There is no multi-host or Swarm-cluster view. Pointing it at a remote daemon means setting `DOCKER_HOST`/TLS env vars before launch; it does not manage connection profiles.
- **Compose coupling is loose and version-sensitive.** Because Compose actions shell out, behavior depends on whether your system has the legacy `docker-compose` (v1, Python) or the `docker compose` plugin (v2). The `command` used is configurable, but a mismatch produces confusing "command not found" or flag errors that look like lazydocker bugs but are environment issues.
- **The socket mount is a privilege boundary.** Running lazydocker in a container (the documented `lazyteam/lazydocker` image) requires bind-mounting `/var/run/docker.sock`, which grants that container effective root on the host. This is fine for local use and a genuine risk on shared or production machines — treat it the same as giving a shell docker-group membership.
- **Pruning is destructive and one keypress away.** The prune actions map to `docker system prune` and friends. They are gated by a confirmation, but the low-friction UI is exactly what makes an accidental volume prune easy; know your keybindings before working against an environment with data you care about.
- **Large environments strain the fixed layout.** With dozens of containers the panels scroll but do not filter richly; the tool is happiest with a handful of services, i.e. a single Compose project, not a machine hosting a hundred containers.
- **Low release cadence.** Point releases arrive every few months. Bugs get fixed eventually rather than urgently; if you need a specific fix, expect to build from `master` or wait. This is normal for a mature single-maintainer tool and should shape expectations, not disqualify it.

## When to Use / When Not

**Use when:**
- You develop locally with Docker/Compose and want logs, stats, restarts, and shells without juggling terminal windows.
- You want a zero-config dashboard that works the moment it is installed.
- You like keyboard-driven TUIs and want per-project custom commands.

**Avoid when:**
- You are operating a cluster (Swarm/Kubernetes) or multiple hosts — this is a single-daemon tool.
- You need scriptable, non-interactive automation; use the `docker` CLI or SDK directly.
- You want a graphical, mouse-first management console for a team — a web UI like Portainer fits better.
- Your target machine is a shared or production host where the socket-access blast radius matters.

## Alternatives

- portainer/portainer — reach for it when you want a web-based, multi-user, multi-host management UI including Swarm/Kubernetes, not a local TUI.
- jesseduffield/lazygit — same author and interaction model, but for Git; the natural companion, not a substitute.
- docker/compose — use the plain `docker compose` CLI when you want scriptable, reproducible operations rather than an interactive dashboard.
- wercc/dry (and similar `dry`-style TUIs) — an alternative terminal UI when you want a lighter or differently-opinioned single-daemon monitor.
- Docker Desktop's dashboard — use it when you are already on Docker Desktop and prefer its GUI container/log views over a terminal.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial commit | 2019-05-18 | Repository created; TUI built on gocui, sibling to lazygit[^5]. |
| v0.1.3 | 2019-06-18 | Earliest tagged release on the releases page[^3]. |
| v0.24.4 | 2026-01-17 | Late-cycle point release[^3]. |
| v0.25.0 | 2026-03-15 | Minor release[^3]. |
| v0.25.2 | 2026-04-19 | Latest release as of this writing; still pre-1.0[^3]. |

## References

[^1]: lazydocker README — "A simple terminal UI for both docker and docker-compose, written in Go with the gocui library." https://github.com/jesseduffield/lazydocker
[^2]: lazydocker README, "Elevator Pitch" section. https://github.com/jesseduffield/lazydocker#elevator-pitch
[^3]: GitHub Releases and Tags for jesseduffield/lazydocker (v0.1.3 … v0.25.2). https://github.com/jesseduffield/lazydocker/releases
[^4]: lazydocker configuration and custom commands documentation. https://github.com/jesseduffield/lazydocker/blob/master/docs/Config.md
[^5]: GitHub repository metadata (created 2019-05-18). https://github.com/jesseduffield/lazydocker

## Tags

go, docker, docker-compose, terminal-ui, tui, devtools, containers, cli, developer-tools, monitoring
