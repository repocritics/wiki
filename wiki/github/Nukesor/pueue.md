# Nukesor/pueue

> A shell command queue with a background daemon — enqueue long-running tasks,
> detach, and manage them from any terminal on the machine.

[GitHub repo](https://github.com/Nukesor/pueue) ·
[crates.io](https://crates.io/crates/pueue) ·
[License: MIT OR Apache-2.0](https://github.com/Nukesor/pueue/blob/main/LICENSE.MIT)

## Overview

Pueue is a Rust command-line tool that processes a queue of shell commands
sequentially or in parallel. It splits into two binaries: `pueue`, the client
you type at, and `pueued`, a daemon that owns the queue and executes tasks as
child processes. Because the daemon is not bound to any terminal, tasks keep
running after you close the SSH session, and any terminal on the same machine
can inspect, pause, reorder, or kill them. The repo dates to 2015; the current
Rust codebase shipped its first release in January 2020[^1].

The project's defining trait is deliberate smallness. The maintainer declares
Pueue **feature-complete**: task dependencies, groups with per-group
parallelism limits, scheduled start times, pause/resume, persisted logs, and
JSON output are in — distributed execution, multi-user support, and
sophisticated scheduling are explicitly out of scope, forever[^2]. The README
states outright that it "hasn't been built for heavy-duty work like hundreds
of tasks." At ~6.3k stars with a single primary maintainer (Arne Beer), it
occupies a stable niche: the human-operated middle ground between `nohup`/tmux
hacks and a real batch scheduler like Slurm.

## Getting Started

```bash
cargo install --locked pueue
# or: your package manager (preferred — ships service files and completions),
# or prebuilt release binaries for Linux/macOS/Windows
```

```bash
pueued -d                                  # start the daemon in the background
pueue add -- rsync -a ./data remote:/backup
pueue add --after 0 -- echo "sync finished"   # runs only after task 0 succeeds
pueue status                               # inspect the queue from any terminal
pueue follow 0                             # tail a running task's output
```

Groups give you independent queues with their own parallelism:

```bash
pueue group add encode
pueue parallel 2 --group encode
pueue add -g encode -- ffmpeg -i in.mkv out.mp4
```

## Architecture / How It Works

The workspace contains three crates: `pueue` (client), `pueued` (daemon), and
`pueue-lib`, which holds the shared state types, settings, and the
client-daemon protocol — and is published separately for anyone who wants
programmatic control of the daemon[^2][^3].

The daemon is the single source of truth. It holds the task list in memory,
spawns tasks as shell child processes in their recorded working directories,
snapshots queue state to disk so it survives crashes and reboots, and writes
each task's output to a per-task log file. The scheduler is intentionally
naive: start queued tasks whenever their group is below its parallel limit and
their dependencies (`--after`) are satisfied. No priority aging, fair-share,
or resource awareness.

Client and daemon talk over a Unix socket on Linux/macOS, or TCP on Windows
and for remote access. TCP connections are protected by TLS with a generated
self-signed certificate plus a pre-shared secret created on first run[^4].
Notable consequences of the design:

- **Environment capture at add-time.** `pueue add` snapshots your current
  environment variables into the task — convenient, but a task added from a
  stale shell runs with stale variables.
- **Everything is shell.** Tasks are strings passed to `sh -c` (or the
  configured shell), enabling pipes and `&&` chains but making escaping the
  user's problem: `pueue add 'ls $HOME && echo done'`.
- **Callbacks.** A templated hook command fires on task completion — the
  standard way to wire up desktop notifications.
- **JSON everywhere.** `pueue status --json`, `pueue log --json`, and
  `format-status` let external scripts consume and re-render queue state.

## Production Notes

- **Client and daemon versions must match.** The protocol and state format are
  not stable across versions; a version mismatch is rejected. Package-manager
  upgrades therefore require a daemon restart — and since tasks are child
  processes of `pueued`, restarting the daemon does not preserve running
  work. Drain the queue (`pueue wait`) before upgrading.
- **The remote-access secret is root-equivalent.** Anyone who can reach the
  TCP port with the shared secret can execute arbitrary shell commands as the
  daemon's user. Treat the secret file like an SSH private key and prefer the
  Unix socket unless you genuinely need remote control[^4].
- **Not a throughput machine.** Status output renders the whole task list and
  state snapshots serialize the whole queue; hundreds of tasks get sluggish.
  Documented as out of scope, not a bug to be fixed[^2].
- **Logs accrue on disk.** Full output of every finished task persists until
  `pueue clean` (or a callback/cron automates it). Chatty tasks (builds,
  ffmpeg) eat surprising space.
- **Feature-complete means slow-moving.** Expect maintenance releases, not
  new capabilities — the upside is genuine stability (last push May 2026).
  v4.0.0 (March 2025) was a breaking overhaul of internals requiring a
  coordinated client/daemon upgrade[^5].
- **Single-user by design.** Per-user daemon, per-user state. Sharing a
  queue across users or machines requires a different tool.

## When to Use / When Not

**Use when:**
- You run long shell jobs (backups, encodes, syncs, builds) over SSH and want
  them to survive disconnects with better ergonomics than `nohup` or tmux.
- You want ordered or dependency-chained commands with bounded parallelism on
  one machine, controlled interactively, with logs and queue state that
  survive reboots — without standing up real infrastructure.

**Avoid when:**
- You need multi-node, multi-user, or resource-aware scheduling — that is
  Slurm/Nomad territory, explicitly out of Pueue's scope.
- You are queueing hundreds of tasks programmatically; Pueue's scheduler and
  status rendering are built for human-scale queues.
- You need supervised, auto-restarting long-lived services — that is a
  process manager's job (pm2, systemd), not a task queue's.

## Alternatives

- leahneukirchen/nq — use instead when you want a job queue with zero daemon
  and zero setup; just flock-based ordering in a directory.
- GNU parallel — use instead when the job is chunking one large batch across
  cores or hosts, rather than tracking many heterogeneous long-running tasks.
- justanhduc/task-spooler — use instead when you want the classic minimal
  Unix task spooler lineage (with GPU-aware queueing in this fork).
- Unitech/pm2 — use instead when the processes are long-lived services to
  supervise and restart, not finite tasks to run once.
- SchedMD/slurm — use instead when you outgrow one machine and need real
  cluster scheduling, partitions, and multi-user accounting.

## History

| Version | Date | Notes |
|---------|------|-------|
| v0.1.0 | 2020-01 | First release of the Rust rewrite (repo dates to 2015)[^1]. |
| v1.0.0 | 2021-08 | First stable release[^5]. |
| v2.0.0 | 2022-02 | Breaking release; config/state changes, coordinated upgrade required[^5]. |
| v3.0.0 | 2022-12 | Breaking release; `pueue-lib` published for programmatic daemon control[^3][^5]. |
| v4.0.0 | 2025-03 | Breaking overhaul of communication layer and internals; project declared feature-complete around this era[^2][^5]. |

## References

[^1]: GitHub API: repository created 2015-09-04; first Rust-era release v0.1.0 published 2020-01-24. https://github.com/Nukesor/pueue/releases
[^2]: Pueue README, "Design Goals" and "What Pueue is not". https://github.com/Nukesor/pueue#design-goals
[^3]: `pueue-lib` on crates.io. https://crates.io/crates/pueue-lib
[^4]: Pueue wiki, "Connect to remote". https://github.com/Nukesor/pueue/wiki/Connect-to-remote
[^5]: Pueue releases page (major version dates). https://github.com/Nukesor/pueue/releases

## Tags

rust, cli, task-queue, daemon, job-queue, process-management, shell, developer-tools, cross-platform, linux
