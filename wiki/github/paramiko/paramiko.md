# paramiko/paramiko

> A pure-Python implementation of the SSHv2 protocol — client and server, and the layer nearly every Python SSH tool is built on.

[GitHub repo](https://github.com/paramiko/paramiko) ·
[Official website](https://www.paramiko.org) ·
[License: LGPL-2.1](https://github.com/paramiko/paramiko/blob/main/LICENSE)

## Overview

Paramiko is a pure-Python implementation of the SSHv2 protocol, providing both client and server functionality[^1]. It has existed since 2003 (originally by Robey Pointer; maintained since the mid-2010s by Jeff Forcier / "bitprophet")[^2], which makes it one of the oldest continuously-used libraries in the Python ecosystem. If a Python program talks SSH, SFTP, or SCP without shelling out to the `ssh` binary, it is almost certainly Paramiko underneath — Ansible, Fabric, pysftp, StackStorm, and large parts of the DevOps tooling world depend on it transitively.

The word "pure-Python" is the defining tension. Paramiko implements the SSH transport, key exchange, channel multiplexing, and SFTP client/server in Python itself; only the cryptographic primitives are delegated to PyCA's `cryptography` library (C/Rust extensions)[^3]. This makes it portable and easy to install, but it also means the protocol state machine runs in Python under the GIL, with a background thread per connection — which is the root of most of its performance and concurrency complaints.

The project's own README steers most users away from calling Paramiko directly: for running remote commands or transferring files, it recommends Fabric (a higher-level wrapper by the same maintainer). Direct Paramiko use is "only intended for users who need advanced/low-level primitives or want to run an in-Python sshd"[^1]. Treat it as infrastructure, not an application-level convenience library.

## Getting Started

```bash
pip install paramiko
```

```python
import paramiko

client = paramiko.SSHClient()
# Load the user's known_hosts; reject unknown hosts by default.
client.load_system_host_keys()
client.set_missing_host_key_policy(paramiko.RejectPolicy())

client.connect("example.com", username="deploy", key_filename="~/.ssh/id_ed25519")

stdin, stdout, stderr = client.exec_command("uname -a")
print(stdout.read().decode())

# SFTP over the same transport
sftp = client.open_sftp()
sftp.put("local.txt", "/remote/local.txt")
sftp.close()

client.close()
```

Do not use `AutoAddPolicy` in production — it silently trusts any host key on first contact, defeating the point of SSH host verification. It appears in nearly every tutorial and is a recurring source of MITM exposure.

## Architecture / How It Works

Paramiko is layered roughly the way the SSH RFCs are:

1. **`Transport`** — owns the socket, runs SSH key exchange, and negotiates ciphers/MACs/host-key algorithms. On `start_client()`/`connect()` it spawns a **background thread** that pumps the protocol: reading packets, handling rekeying, and dispatching to channels. All I/O flows through this one thread.
2. **`Channel`** — a logical stream multiplexed over the transport (a shell, an `exec`, a forwarded port, an SFTP session). Channels implement a file-like API and are flow-controlled by SSH window sizes.
3. **`SFTPClient` / `SFTPServer`** — the SFTP subsystem, built on a channel. The client supports prefetching for reads; writes are pipelined with a configurable number of outstanding requests.
4. **`SSHClient`** — the convenience façade: host-key policy, auth method fallback (key → agent → password), and `exec_command`/`open_sftp`.

Cryptography is not implemented in-house. Since 2.0 (2016) Paramiko delegates ciphers, hashes, and public-key math to PyCA `cryptography`, replacing the older PyCrypto dependency[^4]. Ed25519 and some KEX paths additionally pull in `PyNaCl` and `bcrypt`. This delegation is why Paramiko installs cleanly on most platforms via wheels but still carries a compiled transitive dependency.

The threading model is the load-bearing design decision. Because each `Transport` has its own reader thread and the packet processing runs in Python, throughput is bounded by single-thread Python performance and GIL contention. There is no asyncio-native path; integrating Paramiko into an async application means running it in a thread pool.

## Production Notes

**SFTP is slow by default, and the fix is non-obvious.** Naive `sftp.get()` of a large file can run at a fraction of line rate because it round-trips small requests. Reads improve dramatically with `prefetch=True` (often the default in `get`, but worth verifying) and by reading in large chunks; writes improve by keeping many requests in flight. For bulk transfer of many files, spinning up multiple `Transport`s (not multiple channels on one transport) is the usual scaling lever, because the single reader thread is the bottleneck.

**The OpenSSH 8.8 `ssh-rsa` break bit hard.** OpenSSH 8.8 (2021) disabled the SHA-1 `ssh-rsa` signature algorithm by default. Paramiko versions before 2.9 did not offer the RSA-SHA2 (`rsa-sha2-256`/`512`) signatures, so previously-working RSA key auth suddenly failed with "no matching host key type" or authentication errors against modern servers[^5]. Pin Paramiko ≥ 2.9 if you use RSA keys against current OpenSSH.

**Host-key algorithm negotiation surprises.** Paramiko's host-key handling historically keyed on a single algorithm name, so a server offering both `ssh-ed25519` and RSA could produce confusing "Host key ... not found in known_hosts" errors even when the host was present, if the stored key type differed from what got negotiated. Storing the specific key type the client will negotiate, or using `~/.ssh/known_hosts` semantics carefully, avoids this.

**Concurrency.** A `Transport` and its channels are not a free-threading playground. Sharing one `SSHClient` across many threads doing simultaneous `exec_command` calls leads to interleaving bugs; the safe pattern is one connection per worker. For high fan-out (hundreds of hosts) Paramiko's per-connection thread cost and GIL contention become real; tools like `parallel-ssh` exist specifically to avoid this.

**Large open-issue backlog.** The tracker carries well over a thousand open issues, reflecting both the library's ubiquity and a small maintainer team; response and release cadence are slow and batch-oriented. This is a maturity/stability tradeoff, not abandonment — but do not expect fast turnaround on niche bugs.

**Python 2 is gone.** Paramiko 3.0 (2023) dropped Python 2.7 and older 3.x; code still pinned to Paramiko 2.x for Python 2 compatibility is on an unmaintained branch.

## When to Use / When Not

**Use when:**
- You need programmatic SSH/SFTP inside Python without shelling out to the `ssh` binary.
- You are building a tool that other libraries (Fabric, Ansible-style) sit on top of.
- You need an in-process SSH *server* in Python — Paramiko is one of the few options.
- Portability matters and you want to avoid depending on a system `ssh` install.

**Avoid when:**
- You want an asyncio-native design — reach for `asyncssh` instead.
- You need maximum SFTP/transfer throughput or high host fan-out — a libssh2-backed library is faster.
- Your use case is "run a command / deploy" — use Fabric (higher-level, same maintainer).
- Shelling out to OpenSSH's battle-tested `ssh`/`sftp` binaries is acceptable and simpler for your context.

## Alternatives

- ronf/asyncssh — asyncio-native SSH client and server; use when your app is async or you want cleaner concurrency and generally better throughput.
- fabric/fabric — high-level command execution and deployment layer built on Paramiko; use when you want to run remote commands, not manage protocol primitives.
- ParallelSSH/parallel-ssh — libssh2/ssh2-python backed; use when you need to fan out to many hosts with low overhead.
- ParallelSSH/ssh2-python — thin C `libssh2` binding; use when you want native-speed SSH/SFTP and can accept a compiled dependency.
- OpenSSH (`ssh`/`sftp` via subprocess) — use when a system SSH client is available and you just need to invoke it.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2003 | Original release by Robey Pointer; name = "paranoid" + "friend"[^2]. |
| 1.15 | 2014-09 | Late 1.x line; broad Python 2/3 support era. |
| 2.0 | 2016-04 | Switched crypto backend from PyCrypto to PyCA `cryptography`[^4]. |
| 2.2 | 2017 | Ed25519 key support (via PyNaCl). |
| 2.9 | 2021-12 | RSA-SHA2 signature support; fixes OpenSSH 8.8 `ssh-rsa` breakage[^5]. |
| 3.0 | 2023-01 | Dropped Python 2.7 and older 3.x; modernized baseline[^6]. |
| 3.x | 2023–2026 | Ongoing maintenance line; last push 2026-05[^7]. |

## References

[^1]: Paramiko README — "pure-Python implementation of the SSHv2 protocol ... recommend you use [Fabric]." https://github.com/paramiko/paramiko/blob/main/README.rst
[^2]: Paramiko project site and history; maintainer Jeff Forcier. https://www.paramiko.org
[^3]: Paramiko installation notes — relies on PyCA `cryptography` (C/Rust extensions). https://www.paramiko.org/installing.html
[^4]: PyCA `cryptography`, the crypto backend adopted in Paramiko 2.0. https://cryptography.io
[^5]: OpenSSH 8.8 release notes — disabling `ssh-rsa` (SHA-1) by default. https://www.openssh.com/txt/release-8.8
[^6]: Paramiko changelog. https://www.paramiko.org/changelog.html
[^7]: GitHub API — paramiko/paramiko repository metadata (stars, forks, last push), fetched 2026-07-15. https://github.com/paramiko/paramiko

## Tags

python, ssh, sftp, sshv2, networking, security, cryptography, remote-execution, protocol-library, devops-tooling
