# pexpect/pexpect

> Pure-Python "Expect": spawn an interactive CLI in a pseudo-terminal and drive it by matching output patterns.

[GitHub repo](https://github.com/pexpect/pexpect) ·
[Official website](https://pexpect.readthedocs.io/) ·
[License: ISC](https://github.com/pexpect/pexpect/blob/master/LICENSE)

## Overview

Pexpect automates interactive command-line programs that expect a human at a
terminal — `ssh`, `ftp`, `passwd`, `telnet`, package installers, REPLs. It
launches a child process, then lets your script wait for expected output
patterns and send responses, in the spirit of Don Libes' Tcl `Expect` (1990) but
implemented in pure Python[^1]. Originally written by Noah Spurrier around 2000,
it is now maintained by a small volunteer team and is a foundational dependency
of IPython, Jupyter console, Ansible, and countless CI and provisioning scripts.

The defining mechanism — and the defining tradeoff — is that Pexpect drives
programs through a **pseudo-terminal (pty)**, not through pipes. The child
believes it is talking to a real TTY, so it enables line editing, prompts,
password masking, and color the same way it would interactively. This is what
makes Pexpect able to automate `ssh` and `sudo` where plain `subprocess` cannot.
It is also why the core `spawn` class is **Unix-only**: it depends on the
standard-library `pty` module, which does not exist on Windows[^2].

Pexpect is screen-scraping by design. You are matching regexes against terminal
output meant for human eyes, which is inherently more fragile than a structured
protocol. When a real API exists (SSH via `paramiko`, a REST endpoint, a library
binding), that is almost always the better tool. Pexpect earns its place exactly
when no such interface exists and the only door in is the interactive prompt.

## Getting Started

```bash
pip install pexpect        # pure Python; pulls in ptyprocess on Unix
```

```python
import pexpect

# encoding="utf-8" -> work in str; omit it to work in bytes (the default)
child = pexpect.spawn("ssh user@host", encoding="utf-8", timeout=30)

child.expect("[Pp]assword:")
child.sendline("s3cret")

# Always give expect() the failure cases too, or it raises on timeout/EOF
i = child.expect([pexpect.TIMEOUT, pexpect.EOF, r"\$ "])
if i == 2:
    child.sendline("uptime")
    child.expect(r"\$ ")
    print(child.before)     # text captured *before* the matched prompt

child.sendline("exit")
child.close()
```

## Architecture / How It Works

`pexpect.spawn` forks a child and `execv`s it inside a pty allocated by the
**ptyprocess** package — a companion library the Pexpect authors split out of the
codebase in the 4.0 rewrite so the pty-handling could be reused independently.
The parent keeps the pty master file descriptor and reads from it.

`expect(pattern)` is the core loop. It repeatedly reads a chunk from the pty and
searches a growing internal buffer for the compiled pattern(s). A pattern can be
a regex string, a compiled regex, or the sentinels `pexpect.EOF` and
`pexpect.TIMEOUT`. Passing a **list** matches any of them and returns the index
of the one that matched first — the idiomatic way to branch on prompt vs. error
vs. timeout. After a match, `child.before` holds everything up to the match and
`child.after` holds the matched text itself.

Key surface:

- **`send` / `sendline`** — write to the child. `sendline` appends `\n`.
- **`expect_exact`** — literal-string matching, faster and safer than regex when
  you don't need patterns.
- **`pxssh`** — a `spawn` subclass that encapsulates the SSH login dance
  (prompt detection, password, setting a unique shell prompt) so you don't
  hand-roll it each time.
- **`run()`** — a one-shot convenience for driving a command to completion with
  scripted responses, without managing the object yourself.
- **`fdspawn`** — drive an already-open file descriptor instead of spawning.
- **`PopenSpawn`** (`pexpect.popen_spawn`) — a pipe-backed spawn that works on
  Windows and anywhere a pty is unavailable, at the cost of TTY semantics.

The regex engine is Python's own `re`. Two consequences bite newcomers: `.` does
not match newlines unless you compile with `re.DOTALL`, and `expect()` returns as
soon as the pattern matches *anywhere* in the buffer, so a greedy `.*` or a
missing end-anchor can match far less (or more) than intended.

## Production Notes

**Handle TIMEOUT and EOF explicitly.** By default they raise exceptions. Robust
scripts include them in the `expect` list and branch, rather than wrapping every
call in try/except. A bare `expect(prompt)` that never sees the prompt will hang
until `timeout` then raise — set `timeout` deliberately per call.

**Terminal echo pollutes `before`.** The pty echoes input by default, so the
text you `sendline`d usually appears at the start of the next `before`. Scripts
that parse `before` must account for the echoed command, or disable echo with
`child.setecho(False)` (not honored by every platform/child).

**Line endings.** `sendline` sends `\n`, but a program running under a pty often
expects `\r`. Most shells tolerate `\n`; some raw TTY programs need `child.send("\r")`.

**Large-output performance.** `expect()` re-searches the whole accumulated buffer
on each read, which is O(n²) against a child that dumps megabytes before your
pattern appears. Set `searchwindowsize` to bound the search to the tail of the
buffer, and/or `maxread` to tune chunk size.

**Bytes vs. str.** Without `encoding=`, everything is `bytes` and your patterns
must be byte-strings. Pass `encoding="utf-8"` for `str`; mismatches surface as
`TypeError` on the first `expect`.

**Windows is second-class.** The headline `spawn` does not run on Windows at all;
only `PopenSpawn` and `fdspawn` do, and they lose the pty behavior that is
usually the whole point. Windows users driving real TTY programs generally reach
for the third-party `wexpect` fork instead.

**Fragility.** Because you are matching human-facing output, prompts that change
with locale, shell config, color codes, or program version silently break
scripts. Prefer `expect_exact` and unique sentinel prompts (`pxssh` sets one on
purpose) over matching whatever the default prompt happens to be.

**Maintenance cadence.** Pexpect is mature and low-velocity — the last tagged
release predates the last repository activity by a wide margin, and the open
issue count reflects a long tail on a widely-depended-on library rather than
instability. Treat it as stable-and-done, not actively evolving.

## When to Use / When Not

**Use when:**
- The only interface to a program is its interactive terminal prompt.
- You must automate password/pty-gated tools (`ssh`, `sudo`, `passwd`, `su`).
- You're testing a CLI or REPL end-to-end, exercising real terminal behavior.
- You're on Unix and want scripted install/provisioning without a real API.

**Avoid when:**
- A programmatic API exists — use `paramiko` for SSH, a client library for a service.
- You only need to run a command and read its output: `subprocess` is simpler and sturdier.
- You're primarily on Windows and need true pty semantics.
- The target's prompts/output are unstable across versions or locales.

## Alternatives

- paramiko/paramiko — use for SSH automation (exec, SFTP, port forwarding) instead of screen-scraping the `ssh` CLI.
- pexpect/ptyprocess — use when you want raw pty spawn/read control without any expect-style pattern matching (it's Pexpect's own lower layer).
- fabric/fabric — use for higher-level remote command execution and deployment orchestration built on paramiko.
- amoffat/sh — use for calling non-interactive commands as Python functions with piping; not for prompt-driven interaction.
- Don Libes' Tcl Expect — use when you're already in a Tcl environment or need its mature `interact`/`autoexpect` tooling.

## History

| Version | Date | Notes |
|---------|------|-------|
| ~1.0 | ~2000 | Noah Spurrier's original single-file module. |
| 2.x | 2008–2011 | `pxssh`, `run`, `fdspawn`; widely adopted. |
| 3.0 | 2013-08 | Reorganized into a package; dropped Python 2.5. |
| 4.0 | 2016-01 | Python 3 + Unicode support; pty layer split into `ptyprocess`; `PopenSpawn` for Windows[^3]. |
| 4.6–4.8 | 2017–2019 | Async `expect` (`asyncio`) support, bug fixes. |
| 4.9.0 | 2023-11 | Latest release; maintenance and compatibility fixes[^4]. |

## References

[^1]: Pexpect documentation, "API Overview" / project README. https://pexpect.readthedocs.io/en/stable/overview.html
[^2]: Pexpect docs, "Pexpect on Windows" — notes the pty limitation and `PopenSpawn`/`fdspawn` fallbacks. https://pexpect.readthedocs.io/en/stable/overview.html#pexpect-on-windows
[^3]: `ptyprocess` — pty handling extracted from Pexpect for the 4.0 release. https://github.com/pexpect/ptyprocess
[^4]: Pexpect release history / changelog. https://pexpect.readthedocs.io/en/stable/history.html

## Tags

python, pseudo-terminal, automation, cli-automation, ssh, expect, testing, subprocess, interactive-programs, unix
