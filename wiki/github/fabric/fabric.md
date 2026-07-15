# fabric/fabric

> A Python library for running shell commands remotely over SSH and getting structured results back.

[GitHub repo](https://github.com/fabric/fabric) ·
[Official website](https://fabfile.org) ·
[License: BSD-2-Clause](https://github.com/fabric/fabric/blob/main/LICENSE)

## Overview

Fabric is a Python library for executing shell commands on remote hosts over SSH and capturing their output as Python objects[^1]. It occupies the middle ground between raw `ssh` scripting and full configuration-management systems: you write ordinary Python, call `.run()` on a connection, and get back a result with `stdout`, `stderr`, and exit code. Its most common use is scripted deployments and ad-hoc remote operations ("pull the repo, run migrations, restart the service on these three boxes").

The single most important fact about Fabric is that it exists in two mutually incompatible generations. **Fabric 1.x** (2011–2018) was a monolithic tool centered on a global `env` dictionary, an `@task`-decorated `fabfile.py`, and top-level functions like `run()`, `sudo()`, and `local()` that implicitly acted on "the current host." **Fabric 2.0** (2018) was a ground-up rewrite that threw all of that out: it is built on two lower-level libraries — Invoke[^2] (task execution, CLI parsing, local subprocess handling) and Paramiko[^3] (a pure-Python SSH implementation) — and replaces global state with explicit `Connection` objects. Code written for Fabric 1 does not run on Fabric 2+ without rewriting. Current releases are the 3.x line, which is API-compatible with 2.x but dropped Python 2.7[^4].

This split defines most of the confusion around Fabric. A large body of blog posts, Stack Overflow answers, and legacy `fabfile.py` scripts describe the 1.x API that no longer exists. There is also a separate, unofficial `Fabric3` package on PyPI (a third-party Python-3 port of Fabric 1) that is unrelated to this repository and installs under a conflicting name — a frequent source of broken environments.

## Getting Started

```bash
pip install fabric        # installs the 3.x line; pulls in invoke + paramiko
```

Ad-hoc use from a Python script or REPL — no `fabfile.py` required:

```python
from fabric import Connection

c = Connection("web1.example.com", user="deploy")
result = c.run("uname -s", hide=True)
print(result.stdout.strip())          # e.g. "Linux"

c.put("build.tar.gz", "/srv/app/")    # upload
c.sudo("systemctl restart myapp")     # run as root via sudo
```

Task-style use with a `fabfile.py`, driven by the `fab` CLI:

```python
# fabfile.py
from fabric import task

@task
def deploy(c):
    c.run("git -C /srv/app pull")
    c.run("systemctl --user restart myapp")
```

```bash
fab -H web1.example.com,web2.example.com deploy
```

## Architecture / How It Works

Fabric is deliberately thin. Almost all of its behavior is delegated:

- **Invoke** provides the task system (`@task`, the `fab`/`inv` CLI, argument parsing), the `Context` abstraction, `Result` objects, and local subprocess execution (`Context.run`). Fabric's `@task` and `Connection` are subclasses/extensions of Invoke primitives.
- **Paramiko** implements the SSH2 protocol in pure Python: transport, authentication, channels, and SFTP. Fabric's `Connection` wraps a Paramiko `SSHClient`.

The central object is **`Connection`** — one SSH connection to one host. Its methods map onto remote operations: `.run()` (execute a command), `.sudo()` (run under `sudo` with password prompt handling), `.get()` / `.put()` (SFTP transfer), `.local()` (delegated to Invoke, runs on the client), and `.forward_local()` / `.forward_remote()` for port forwarding. Every call returns a `Result` (or raises on non-zero exit unless `warn=True`).

For multiple hosts there are **`Group`** classes — `SerialGroup` (one host at a time) and `ThreadingGroup` (concurrent via a thread pool). There is no global "list of hosts" as in Fabric 1; you construct groups explicitly, and results come back as a dict-like `GroupResult` keyed by connection.

Configuration is layered through Invoke's `Config` system, extended by Fabric to understand SSH concerns: system and user YAML files (`/etc/fabric.yaml`, `~/.fabric.yaml`), environment variables, per-`Connection` overrides, and — importantly — your existing OpenSSH client config (`~/.ssh/config`) when `use_ssh_config` is enabled, which lets `HostName`, `User`, `Port`, `IdentityFile`, and `ProxyJump`/gateway directives be honored. Jump-host chaining is supported through the `gateway` argument or SSH-config `ProxyJump`.

The design tradeoff is explicit statefulness over convenience. Fabric 1's globals made short scripts terse but made larger programs hard to reason about and impossible to use as a normal importable library. Fabric 2+ is more verbose but composes like ordinary Python.

## Production Notes

**Paramiko is the practical ceiling.** Because SSH is implemented in pure Python rather than shelling out to OpenSSH, Fabric inherits Paramiko's performance characteristics and its historically slower adoption of new crypto. The most notorious real-world incident: OpenSSH 8.8 (2021) disabled the `ssh-rsa` (SHA-1) signature algorithm by default, and older Paramiko versions could not negotiate the RSA-SHA2 replacement — producing sudden "authentication failed" errors against freshly upgraded servers. Keep Paramiko current, and prefer Ed25519 keys. Any exotic cipher, KEX, or hardware-token (`GSSAPI`, FIDO/`sk-` keys) requirement should be validated against Paramiko's support matrix before committing to Fabric.

**Concurrency is threads, not async.** `ThreadingGroup` fans out over a thread pool. This is fine for I/O-bound SSH work, but there is no event-loop model, no built-in rate limiting, and failures in one host's thread must be handled through `GroupResult` inspection. For hundreds-to-thousands of hosts, Fabric is workable but unremarkable; tools built for fleet scale (Ansible with forks, or SSH multiplexers) are better fits.

**No idempotency, no state, no facts.** Fabric runs imperative commands. Re-running a deploy task re-runs every command; there is no notion of "already in desired state." Anything resembling configuration management (package convergence, templated files, handlers) is your responsibility to build. This is the main axis on which Fabric and Ansible are not substitutes.

**The 1→2 migration is a rewrite, not an upgrade.** There is no compatibility shim. `env.hosts`, `@parallel`, `roles`, `execute()`, top-level `run()`/`sudo()`/`local()`, and the entire `fabric.api` module are gone. Plan the port as new code. Upgrade documentation exists but does not automate the change[^5].

**Watch the PyPI name collision.** `pip install fabric` is correct. `pip install Fabric3` installs an unrelated third-party fork of the old 1.x codebase; having both in an environment produces import-shadowing bugs. Pin `fabric>=2` in requirements to avoid resolvers pulling legacy packages.

**`sudo` password handling.** `Connection.sudo()` needs the sudo password, sourced from config (`sudo.password`), a prompt responder, or `--prompt-for-sudo-password` on the CLI. Passwordless sudo on the target simplifies automation but is a security decision, not a Fabric one.

## When to Use / When Not

**Use when:**
- You want to script SSH operations in real Python and get structured results back.
- Your task is imperative deployment or ops glue over a handful to a few dozen hosts.
- You already lean on OpenSSH config (jump hosts, per-host identities) and want it honored.
- You need Fabric embedded as a library inside a larger Python program, not just a CLI.

**Avoid when:**
- You need declarative, idempotent, inventory-driven config management — reach for Ansible.
- You're orchestrating containers/Kubernetes rather than long-lived SSH hosts.
- You require an async model or very-large-fleet concurrency with backpressure.
- Your existing scripts are Fabric 1.x and you cannot afford the rewrite to 2+.

## Alternatives

- ansible/ansible — declarative, idempotent, agentless config management; use instead when you need convergence, inventory, and modules rather than raw commands.
- pyinvoke/invoke — Fabric's own foundation; use instead when the work is purely local task-running with no SSH.
- paramiko/paramiko — the SSH layer underneath Fabric; use instead when you need low-level protocol control and don't want Fabric's task/connection conventions.
- saltstack/salt — event-driven remote execution and config management at scale; use instead for large fleets with a control plane.
- mitogen — fast Python-over-SSH bootstrapping; use instead when raw remote-Python execution speed is the priority.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.1 | 2009 | Initial public release as a standalone SSH deployment tool[^1]. |
| 1.0 | 2011-03 | First stable release; `fabfile.py` + global `env` API, Python 2 only. |
| 2.0.0 | 2018-05-10 | Ground-up rewrite on Invoke + Paramiko; `Connection` objects, no globals; not backward compatible[^2]. |
| 2.x | 2018–2022 | Groups, config layering, SSH-config integration matured across 2.1–2.7. |
| 3.0.0 | 2023-01-20 | Dropped Python 2.7; modernized dependency floors[^4]. |
| 3.2.3 | 2026-04-05 | Latest 3.x maintenance release. |

## References

[^1]: Fabric documentation — "Welcome to Fabric!". https://www.fabfile.org/
[^2]: Invoke — subprocess execution and CLI toolkit that Fabric builds on. https://www.pyinvoke.org/
[^3]: Paramiko — pure-Python SSHv2 protocol library used by Fabric for transport and SFTP. https://www.paramiko.org/
[^4]: Fabric changelog. https://www.fabfile.org/changelog.html
[^5]: Fabric docs — "Upgrading from 1.x". https://www.fabfile.org/upgrading.html

## Tags

python, ssh, deployment, remote-execution, devops, automation, paramiko, invoke, cli, sysadmin
