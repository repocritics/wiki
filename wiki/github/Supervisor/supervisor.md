# Supervisor/supervisor

> A client/server process control system for UNIX — keeps a fixed set of foreground programs running and restarts them when they die.

[GitHub repo](https://github.com/Supervisor/supervisor) ·
[Official website](http://supervisord.org) ·
License: BSD-derived (custom, see repo `LICENSES.txt`)[^1]

## Overview

Supervisor is a long-lived daemon (`supervisord`) that starts, monitors, and
restarts a configured list of child processes on UNIX-like systems. It was
written by Chris McDonough and collaborators and predates most modern process
managers; the codebase moved to GitHub in 2010 but the project itself dates to
the mid-2000s[^2]. Its design goal is narrow and unchanged: take a set of
line-oriented programs, run each in the foreground, capture their stdout/stderr,
and bring them back up if they exit.

The defining constraint — and the source of most confusion — is that Supervisor
controls **foreground** processes only. It does not read PID files and it does
not talk to the operating system's service manager. It supervises processes it
`fork()`/`exec()`s itself and tracks their life via `SIGCHLD`. A program that
daemonizes (double-forks and detaches) will appear to Supervisor to have exited
immediately, and Supervisor will restart it in a loop. Every deployment mistake
in Supervisor traces back to this rule.

Supervisor is not an init system. It cannot be PID 1 and does not manage the
boot sequence; you still need `systemd`, an rc script, or a container entrypoint
to launch `supervisord` itself. It occupies the layer above the OS init and
below the application: "run these five worker processes and keep them alive,"
without the complexity of a full service manager.

## Getting Started

```bash
pip install supervisor
echo_supervisord_conf > /etc/supervisord.conf   # emit a sample config
```

```ini
; /etc/supervisord.conf
[supervisord]
nodaemon=false
logfile=/var/log/supervisord.log

[program:worker]
command=/usr/bin/python3 /app/worker.py   ; MUST run in the foreground
directory=/app
autostart=true
autorestart=true
startsecs=5                                ; must stay up 5s to be "running"
startretries=3
stdout_logfile=/var/log/worker.log
stderr_logfile=/var/log/worker.err
numprocs=1
user=appuser

[supervisorctl]
serverurl=unix:///var/run/supervisor.sock
```

```bash
supervisord -c /etc/supervisord.conf   # start the daemon
supervisorctl status                   # worker  RUNNING  pid 4123, uptime 0:01:12
supervisorctl restart worker
supervisorctl reread && supervisorctl update   # apply config changes
```

## Architecture / How It Works

Supervisor is three cooperating parts:

1. **`supervisord`** — the server. Parses the INI config, spawns each
   `[program:x]` as a child, and runs an event loop that reaps `SIGCHLD`,
   applies restart policy, and rotates captured logs.
2. **`supervisorctl`** — a shell/CLI client that talks to the daemon over a
   UNIX domain socket (or TCP) using XML-RPC. `status`, `start`, `stop`,
   `tail`, and `reload` are all XML-RPC calls under the hood.
3. **XML-RPC / HTTP interface** — the same RPC surface, optionally exposed over
   `inet_http_server` with an included web UI. Third-party tooling drives
   Supervisor through this API.

Process state is a small state machine: `STOPPED → STARTING → RUNNING`, with
`BACKOFF`, `EXITED`, `FATAL`, and `STOPPING` for the failure and shutdown paths.
A process is only considered `RUNNING` once it has stayed alive for `startsecs`;
a program that crashes faster than that cycles `STARTING → BACKOFF` until
`startretries` is exhausted, then lands in `FATAL` and is left alone.

Grouping is done with `numprocs` (which templates `program_name_%(process_num)s`)
and `[group:x]` sections. Startup and shutdown order is controlled only by an
integer `priority` — there is no real dependency graph, no "start B after A is
healthy." The **event listener** subsystem (`[eventlistener:x]`) lets a process
subscribe to events like `PROCESS_STATE`, `TICK_60`, or `PROCESS_LOG` over a
simple stdin/stdout protocol; the bundled `memmon` listener (from the
`superlance` package) uses this to restart processes that exceed a memory
threshold.

## Production Notes

- **Foreground only, always.** `command=` must not daemonize. Disable any
  `--daemon`/`--detach`/`fork` flag. If a program insists on backgrounding,
  Supervisor loses track of it and restart/stop semantics break.
- **`reread`/`update` vs `restart`.** Editing the config does nothing until you
  `supervisorctl reread` then `update`. `restart` only bounces the process with
  its *current* config. This trips up nearly every new operator.
- **Signals and slow shutdown.** Stop is `stopsignal` (default `TERM`) followed
  by `SIGKILL` after `stopwaitsecs`. Apps that trap `TERM` or spawn a subshell
  (`command=bash -c "..."`) can leave orphaned children, because the signal goes
  to the shell, not the real process. Prefer `exec` in wrapper scripts.
- **No resource isolation.** Supervisor has no cgroups, CPU, or memory limits of
  its own. For real limits, run `supervisord` under `systemd` with a slice, or
  use `superlance`'s `memmon` for a coarse memory-based restart.
- **XML-RPC is an attack surface.** CVE-2017-11610 allowed remote command
  execution through the XML-RPC interface's nested `supervisor.supervisord`
  namespace[^3]. Never expose `inet_http_server` without authentication and a
  network boundary; a UNIX socket is the safe default.
- **Log rotation is built in but per-file.** `stdout_logfile_maxbytes` +
  `stdout_logfile_backups` rotate captured output. Set `maxbytes=0` to disable
  and hand logging to the app, or logs will grow to the configured cap on disk.
- **Containers.** Running Supervisor as a container's PID 1 to manage multiple
  processes works but fights the "one process per container" model and can
  swallow exit codes and signals. It is a pragmatic escape hatch, not a
  recommendation.

## When to Use / When Not

**Use when:**
- You need to keep a handful of foreground worker/daemon processes alive on a
  plain VM or bare-metal host without writing systemd units per service.
- You want centralized start/stop/restart and captured logs across processes
  with a single INI file and one CLI.
- You are on a system without systemd, or want the same config to work across
  several UNIX flavors.

**Avoid when:**
- You target Windows — Supervisor does not run on it at all[^4].
- The host already runs `systemd` and you want cgroup limits, socket activation,
  boot ordering, and dependency management — use native units instead.
- You need per-service resource isolation, health-check-based dependency
  ordering, or a scheduler; reach for systemd, a container orchestrator, or s6.

## Alternatives

- systemd — on modern Linux, native units give cgroup limits, dependency
  ordering, and boot integration Supervisor lacks; use it when systemd is present.
- circus (Mozilla) — Python process/socket manager with ZeroMQ control and
  built-in socket sharing; use when you need socket activation from a Python tool.
- s6 / skarnet s6 — minimal, correct process supervision suite; use as a
  container PID 1 where signal/reaping correctness matters most.
- runit — tiny per-service `run` scripts and fast reliable restarts; use for
  a lightweight, filesystem-driven supervision tree.
- Unix4fun/immortal or foreman/honcho — use for dev-time Procfile-style running
  rather than production supervision.

## History

| Version | Date | Notes |
|---------|------|-------|
| 2.x | 2004–2007 | Early releases under the repoze umbrella; Python 2 only. |
| 3.0 | 2013-07 | First 3.x stable; XML-RPC + web UI, Python 2 only[^2]. |
| 3.3.3 | 2017-07 | Security fix for CVE-2017-11610 (XML-RPC RCE)[^3]. |
| 4.0.0 | 2019-04 | Python 3 support (3.4+) added alongside Python 2.7[^5]. |
| 4.2.x | 2020–2022 | Config/expansion fixes; 4.2.5 (2022) a common pinned baseline. |

## References

[^1]: Supervisor license terms (BSD-derived, custom). GitHub reports the SPDX id as NOASSERTION; see `LICENSES.txt` in the repository. https://github.com/Supervisor/supervisor/blob/main/LICENSES.txt
[^2]: Supervisor documentation and repository history. http://supervisord.org/
[^3]: CVE-2017-11610 — Supervisor XML-RPC remote command execution; fixed in 3.0.1/3.1.4/3.2.4/3.3.3. https://nvd.nist.gov/vuln/detail/CVE-2017-11610
[^4]: Supervisor README, "Supervisor will not run at all under any version of Windows." https://github.com/Supervisor/supervisor/blob/main/README.rst
[^5]: Supervisor changelog / 4.0.0 release notes. http://supervisord.org/changes.html

## Tags

python, process-manager, process-supervisor, unix, daemon, devops, sysadmin, supervisord, xml-rpc, service-management, cli
