# puma/puma

> A multi-threaded, pre-forking HTTP/1.1 server for Ruby/Rack apps — the default server for Rails.

[GitHub repo](https://github.com/puma/puma) ·
[Official website](https://puma.io) ·
[License: BSD-3-Clause](https://github.com/puma/puma/blob/main/LICENSE)

## Overview

Puma is a Rack-compatible application server for Ruby, originally written by
Evan Phoenix and first targeted at Rubinius before becoming the de facto server
for MRI (CRuby) and JRuby deployments[^1]. Since Rails 5 (2016) it has shipped
in the default generated `Gemfile`, which makes it the server most Rails apps
run in production whether or not the team chose it deliberately[^2].

Its defining design choice is combining two concurrency models at once:
multiple OS processes ("workers", via `fork`) and a thread pool inside each
worker. This is a direct response to Ruby's runtime reality. On MRI the Global
VM Lock (GVL) allows only one thread to execute Ruby bytecode at a time, so
threads help only when requests spend time blocked on IO (database calls,
external HTTP, disk). Truly parallel runtimes — JRuby, TruffleRuby — run threads
in parallel without the GVL. Puma's config surface therefore has no single
"right" setting: the correct thread and worker counts depend on whether your
workload is IO-bound or CPU-bound and which Ruby you run[^3]. That tuning burden,
plus the requirement that your entire app and every gem be thread-safe, is the
central tension of running Puma.

## Getting Started

```bash
gem install puma
puma          # looks for ./config.ru
```

```ruby
# config/puma.rb — typical Rails production config
workers Integer(ENV.fetch("WEB_CONCURRENCY", 2))   # OS processes
threads_count = Integer(ENV.fetch("RAILS_MAX_THREADS", 5))
threads threads_count, threads_count               # min:max per worker

preload_app!                                       # enable copy-on-write

bind "tcp://0.0.0.0:9292"

on_worker_boot do
  ActiveRecord::Base.establish_connection          # reconnect after fork
end
```

```bash
bundle exec puma -C config/puma.rb   # prefer the executable over `rails server`
```

## Architecture / How It Works

Puma has three moving parts inside a running server:

1. **Cluster master** — in cluster mode (`workers N > 0`) a master process
   `fork`s N worker children and supervises them. With `preload_app!` the app is
   loaded once in the master before forking, so unchanged pages of memory are
   shared via copy-on-write until a child writes to them. The master never serves
   requests; it monitors workers and coordinates restarts.
2. **Thread pool** — each worker runs a pool of request-handling threads
   (`threads min, max`). The pool scales between min and max with load. The `-t`
   count is per worker, so `-w 2 -t 5:5` yields ten request threads total.
3. **Reactor** — a dedicated thread using non-blocking IO buffers slow/partial
   requests off the pool. Puma reads the full request before dispatching it to an
   app thread, which is why it does not strictly require an upstream proxy to
   protect against slow-client (Slowloris-style) attacks the way Unicorn does.

The HTTP/1.1 parser is a C extension inherited from Mongrel, with well over a
decade of production exposure[^1]. Restarts come in three flavors: **hot restart**
(SIGUSR2, keeps the listening socket open so no connections drop — unavailable on
JRuby/Windows, which cannot pass descriptors to a new process), **phased restart**
(SIGUSR1, rolls workers one at a time for zero-downtime deploys, but is
incompatible with `preload_app!`), and plain stop/start. A `fork_worker` mode lets
worker 0 fork grandchildren to refresh copy-on-write savings over long uptimes.

A built-in control/status app (`--control-url` + token) exposes stats and
lifecycle commands, driven by the `pumactl` CLI.

## Production Notes

- **Thread-safety is on you.** Puma's whole value proposition assumes your app,
  and every gem it loads, is thread-safe. A non-thread-safe dependency produces
  intermittent, load-dependent corruption that will not show up in single-thread
  testing. This is the single most important migration consideration versus a
  process-only server.
- **Size the DB pool to the thread count.** ActiveRecord's connection pool must be
  ≥ the max thread count or threads will block waiting for connections; the common
  convention is to drive both from `RAILS_MAX_THREADS`.
- **Memory fragmentation is real.** Long-lived multi-threaded MRI workers suffer
  glibc `malloc` arena fragmentation. Two standard mitigations: set
  `MALLOC_ARENA_MAX=2`, or link jemalloc. Many teams also run the external
  `puma_worker_killer` gem to recycle workers on a memory ceiling.
- **`preload_app!` and forking sockets.** Any socket opened in the master before
  fork (DB, Redis, HTTP keep-alives) is copied into every child and will produce
  `Errno::EPIPE`/`EOFError` if used concurrently. Reconnect inside `on_worker_boot`
  / `before_worker_boot`; background threads from third-party gems are *not*
  inherited across `fork` and must be restarted.
- **Kubernetes graceful shutdown is a known footgun.** K8s pod termination
  routing interacts badly with Puma's graceful drain; requests can arrive after
  the server begins shutting down. Puma ships dedicated Kubernetes docs for this[^4].
- **CPU-bound work on MRI does not parallelize.** Adding threads to a CPU-bound
  MRI app increases GVL contention without adding throughput; scale with workers
  (processes) instead, or move to JRuby/TruffleRuby.
- **`rails server` hides options.** The Rails launcher ignores much of Puma's
  config surface; run `bundle exec puma -C config/puma.rb` for real deployments.
- **Daemonization was removed in Puma 5.** The `daemonize` option is gone; use
  systemd, rc.d, or a process supervisor (or the third-party `puma-daemon` gem on
  MRI)[^5].

## When to Use / When Not

**Use when:**
- You run Rails/Rack and want the supported, default path.
- Your workload is IO-bound (DB, external APIs) — threads reclaim GVL idle time.
- You want processes and threads together to trade memory against concurrency.
- You want zero-downtime deploys (phased/hot restart) without an external tool.

**Avoid / reconsider when:**
- Your app or a critical gem is not thread-safe — a single-threaded pre-fork
  server (Unicorn) or a hybrid (Passenger) sidesteps the class of bug entirely.
- Your workload is CPU-bound pure Ruby on MRI — threads add contention, not speed.
- You want a fully async, fiber-per-request model for very high IO concurrency —
  Falcon fits that shape better.

## Alternatives

- socketry/falcon — use instead when you want a fiber/async, non-blocking server
  built for high IO concurrency on modern Ruby.
- phusion/passenger — use instead when you want an integrated ops experience
  (nginx/Apache integration, hybrid multi-process model, monitoring built in).
- macournoyer/thin — use instead for a small EventMachine-based server for simple
  or legacy single-threaded services.
- ruby/webrick — use instead only for local development or tiny scripts; not a
  production server. (Unicorn — single-threaded pre-fork — is the other classic
  production comparison for non-thread-safe apps.)

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2011-09 | Repository created; server for Rubinius[^1]. |
| default in Rails | 2016 | Puma becomes the default Rails app server[^2]. |
| 4.0 | 2019-06 | Persistent connection handling, low-level improvements. |
| 5.0 | 2020-10 | `daemonize` removed; `fork_worker`, `wait_for_less_busy_worker`, `nakayoshi_fork`[^5]. |
| 6.0 | 2022-11 | Rack 3 compatibility, control-app/status improvements, internal cleanup. |

(Exact minor-version dates omitted where not independently verified; see the
CHANGELOG for the authoritative history[^5].)

## References

[^1]: Puma README — "A Ruby Web Server Built For Parallelism"; HTTP parser inherited from Mongrel, origin as a Rubinius server. https://github.com/puma/puma/blob/main/README.md
[^2]: Puma README, "Frameworks / Rails" — Puma is the default server for Rails, included in the generated Gemfile. https://github.com/puma/puma/blob/main/README.md
[^3]: Puma deployment docs — thread/worker count tradeoffs and the MRI GVL. https://github.com/puma/puma/blob/main/docs/deployment.md
[^4]: Puma Kubernetes docs — graceful shutdown interaction. https://github.com/puma/puma/blob/main/docs/kubernetes.md
[^5]: Puma CHANGELOG / History — `daemonize` removal and cluster-mode features. https://github.com/puma/puma/blob/main/History.md

## Tags

ruby, rack, http-server, web-server, multithreading, cluster-mode, rails, concurrency, application-server, prefork
