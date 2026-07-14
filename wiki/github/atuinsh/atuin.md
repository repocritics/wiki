# atuinsh/atuin

> Replaces your shell history with an indexed SQLite database and optional end-to-end-encrypted sync across machines.

[GitHub repo](https://github.com/atuinsh/atuin) ·
[Official website](https://atuin.sh) ·
[License: MIT](https://github.com/atuinsh/atuin/blob/main/LICENSE)

## Overview

Atuin, started by Ellie Huxtable in 2020[^1], replaces the plain-text
`~/.bash_history` / `~/.zsh_history` file with a SQLite database, and rebinds
`Ctrl-R` (and, by default, the up arrow) to a full-screen fuzzy search UI over
that database. Beyond the command text, it records structured context per
entry: exit code, working directory, hostname, session id, timestamp, and
command duration[^2]. That turns shell history from an append-only text log
into a queryable dataset — `atuin search --exit 0 --after "yesterday 3pm" make`
is a first-class query, not a `grep` incantation.

The second half of Atuin is **sync**: an optional server that replicates history
between machines. History is end-to-end encrypted with a key that never leaves
your machines, so neither the project's hosted "Atuin Cloud" nor a self-hosted
instance can read command contents[^3]. Sync is opt-in — Atuin is fully usable
as a local-only history tool with no account.

The defining tension is scope creep against a very intimate surface. Shell
history is something every developer touches hundreds of times a day, and
Atuin inserts itself into the `preexec`/`precmd` hot path of that interaction.
The payoff (fast contextual search, cross-machine continuity) is real, but the
cost is a Rust binary, a database, shell hooks, and — if you enable sync — a
key you must never lose. The project has since expanded further into Atuin
Desktop, a runbook product[^5], which widens the umbrella well past "shell
history."

## Getting Started

```bash
# installer sets up the binary and the shell plugin
curl --proto '=https' --tlsv1.2 -LsSf https://setup.atuin.sh | sh
```

Local-only use needs no account — just import existing history and restart the
shell:

```bash
atuin import auto      # ingest your existing HISTFILE
exec $SHELL            # reload so the Ctrl-R hook is active
```

Optional encrypted sync (hosted or self-hosted):

```bash
atuin register -u <USERNAME> -e <EMAIL>   # or `atuin login`
atuin key                                 # PRINT AND BACK UP — unrecoverable if lost
atuin sync
```

Query from the CLI without the interactive UI:

```bash
# most-used commands in the current directory
atuin stats
atuin search --cwd . --limit 20
```

## Architecture / How It Works

Atuin is a Rust workspace split into a client, a server, and shared crates.
The pieces that matter operationally:

- **Local store.** History lives in a SQLite database (by default under
  `~/.local/share/atuin/`), separate from — not a replacement for — your
  existing `HISTFILE`, which Atuin keeps writing so uninstalling is non-lossy.
  Config is TOML at `~/.config/atuin/config.toml`.
- **Shell integration.** Each shell records commands via its lifecycle hooks:
  `preexec`/`precmd` on zsh, the equivalent on fish and nushell, and — the
  awkward case — `bash-preexec` on bash, since bash has no native preexec[^4].
  The hook captures start time, exit code, cwd, and duration and writes a row.
- **Search UI.** The interactive TUI queries SQLite live. **Filter modes**
  (global / host / session / directory / workspace) scope results, and are
  cycled from inside the search with `Ctrl-R`.
- **Sync.** History rows are encrypted client-side and pushed to a server as
  an append-only log. The current design is the **"record" sync** protocol
  (sometimes called sync v2), a generic encrypted-record store that superseded
  the original history-specific v1 sync and also backs features like dotfile
  (alias/env) sync. The server is Rust backed by PostgreSQL.

The coupling story: Atuin sits *between* you and every command you run. That
hot-path position is why shell-hook edge cases (bash especially) dominate the
issue tracker, and why the encryption key is load-bearing — the server is
deliberately blind, so key loss is unrecoverable by design, not a bug.

## Production Notes

**Bash is the weak shell.** zsh and fish get native-quality integration; bash
relies on `bash-preexec`, which has known limitations and conflicts with other
tools that also hook `PROMPT_COMMAND`/`DEBUG` trap (fzf's own bindings, other
preexec consumers)[^4]. The project increasingly recommends `ble.sh` for a
better bash experience. Budget time here on bash-heavy fleets.

**The up-arrow rebind surprises people.** By default Atuin captures the up
arrow to open its search UI, which changes decades of muscle memory (up = the
literal previous line). Many users disable this via
`enter_accept` / keybinding config; it is the single most common "why is my
shell weird now" complaint. Decide the team default before rolling out.

**Back up the key.** `atuin key` prints the symmetric key. If you enable sync,
lose the key, and lose every machine that had it, the encrypted history on the
server is permanently unreadable — the server cannot help you. Store the key
in your password manager the moment you register.

**Sync protocol skew.** The move from v1 to record sync means client and
server versions matter. A very old client against a newer server (or vice
versa) can fail to sync cleanly; self-hosters must keep the server reasonably
current. Self-hosting also means running and backing up a PostgreSQL instance —
non-trivial if you only wanted shell history.

**Database growth and imports.** `atuin import auto` on a large, years-old
`HISTFILE` can pull in a lot of noise (and duplicates from multiple shells).
Interactive search stays fast on large DBs thanks to SQLite indexing, but the
initial import and stats over huge datasets are the slow paths.

**It does not replace secrets hygiene.** Commands with inline secrets are
recorded like any other. There is history filtering config
(`history_filter` regexes) to exclude patterns, but it must be set up
deliberately — the default is to capture everything.

## When to Use / When Not

**Use when:**
- You live in the shell and want contextual, fuzzy, exit-code-aware history
  search instead of `Ctrl-R` linear scrollback.
- You work across multiple machines and want the same history everywhere,
  without trusting a third party with plaintext.
- You want history as data — stats, per-directory recall, scripting over past
  commands.

**Avoid when:**
- Your fleet is bash-first and you can't invest in `bash-preexec`/`ble.sh`
  tuning.
- You want zero moving parts — a plain `HISTFILE` plus `fzf` is lighter if you
  only need fuzzy search and no sync.
- You're uneasy about a tool in the preexec hot path, or can't guarantee the
  encryption key will be safely retained across machine loss.

## Alternatives

- cantino/mcfly — Rust, "neural" ranking of history by context; search-only,
  no sync. Use when you want smarter ranking without a server or database story.
- junegunn/fzf — the minimal option: fuzzy `Ctrl-R` over your existing
  `HISTFILE`, no database, no context, no sync. Use when you only want fuzzy
  search and nothing else.
- curusarn/resh — records context (cwd, exit code, duration) like Atuin but
  stays local. Use when you want rich context without any sync/server.
- dvorka/hstr — C, lightweight interactive history filtering. Use for a tiny,
  dependency-light TUI over plain history.
- zsh-users/zsh-history-substring-search — zsh plugin for substring recall
  only. Use when you're zsh-only and want a minimal keybinding, not a tool.

## History

| Version | Date | Notes |
|---------|------|-------|
| — | 2020-10 | Project started by Ellie Huxtable[^1]. |
| record sync (v2) | ~2023 | Generic encrypted-record sync replacing history-only v1; also backs dotfile sync. |
| Atuin Desktop | 2025 | Runbook/desktop product announced under the Atuin umbrella[^5]. |

## References

[^1]: atuinsh/atuin repository and author (Ellie Huxtable). https://github.com/atuinsh/atuin
[^2]: Atuin documentation — basic usage and recorded fields. https://docs.atuin.sh/guide/basic-usage/
[^3]: Atuin documentation — sync and end-to-end encryption. https://docs.atuin.sh/guide/sync/
[^4]: Atuin docs — shell plugin / bash (`bash-preexec`) installation notes. https://docs.atuin.sh/guide/installation/
[^5]: Atuin Desktop. https://atuin.sh/

## Tags

rust, shell, shell-history, cli, terminal, sqlite, sync, end-to-end-encryption, developer-tools, zsh, bash, fish
