# tstack/lnav

> A terminal log file viewer that merges, indexes, and lets you query logs with SQL.

[GitHub repo](https://github.com/tstack/lnav) ·
[Official website](https://lnav.org) ·
[License: BSD-2-Clause](https://github.com/tstack/lnav/blob/master/LICENSE)

## Overview

lnav (the Logfile Navigator) is a single-binary TUI for reading logs, written in C++ by Timothy Stack. The repository dates back to 2009[^1], and the project is still actively maintained, with commits landing in 2026. Point it at files or directories and it decompresses what it needs (gzip, bzip2, and archives via libarchive), detects each file's format, and merges everything into one view ordered by timestamp. That time-merge across heterogeneous files is the core value proposition — `tail -f a.log b.log` interleaves lines as they arrive, while lnav interleaves them by the time they were actually logged, and lets you scroll backward.

The defining design decision is that lnav treats logs as a database rather than as text. Every recognized log message becomes a row in a SQLite virtual table, so you can filter and aggregate with real SQL (`;` in the TUI), not just grep. This is also the source of its main tension: the SQL/analytics power and the semantic highlighting only apply to logs whose format lnav recognizes. Lines it cannot parse fall back to a plain TEXT view with none of the indexing, error-jumping, or column-aware querying. Getting value out of a nonstandard format means writing a JSON format definition, which is a real (if one-time) cost.

lnav is aimed at operators, developers, and anyone debugging on a box where shipping logs to a centralized stack (Loki, Elasticsearch, Splunk) is overkill or unavailable. It is a local, single-host tool: there is no server, no agent, no index on disk to manage.

## Getting Started

```console
$ brew install lnav          # macOS
$ pkg install lnav           # FreeBSD
# or download a static binary from the GitHub releases page for Linux/macOS/Windows
```

Point it at files or directories; it figures out the rest:

```console
$ lnav /var/log/syslog /var/log/nginx/
```

Pipe `systemd-journald` output in as a pager (use `-o short-iso` so the year is
present, otherwise timestamp parsing gets confused across year boundaries):

```console
$ journalctl -o short-iso | lnav
```

Inside the TUI, drop into SQL with `;` and query the merged log stream:

```sql
SELECT log_time, log_level, log_body
FROM all_logs
WHERE log_level >= 'error'
ORDER BY log_time DESC
LIMIT 20;
```

## Architecture / How It Works

lnav's pipeline is: **discover → decompress → detect format → index → merge → present**. Each stage is worth understanding because each is also where things break.

- **Format detection.** lnav ships with built-in format definitions (syslog, common web access logs, many JSON-lines shapes, etc.) and matches each file against them by trying to parse the timestamp and structural fields. Detection is per-file and can be ambiguous; a file that half-matches a format can be mis-classified. Custom formats are JSON files dropped in the config directory that declare a regex to extract the timestamp, level, and named fields.
- **Indexing.** For each recognized message lnav records its byte offset, timestamp, and log level, building an in-memory index. Multi-line messages (stack traces, pretty-printed JSON) are folded into a single logical message. The error/warning index is what powers instant jump-to-next-error (`e`/`E`).
- **Time merge.** All files' messages are merged into one monotonic time order. Renames are followed, new files appearing in a watched directory are picked up, and files are tailed live.
- **SQLite layer.** Messages are exposed as virtual tables. Beyond the generic `all_logs`, each format contributes columns for its parsed fields, so you can `SELECT`, `WHERE`, `GROUP BY`, and `JOIN` over structured log data. Recent versions also accept PRQL as an alternative query language, which is why the build depends on a Rust toolchain to compile the PRQL compiler.
- **Presentation.** Semantic highlighting assigns stable colors to identifiers (IPs, PIDs), errors render red, and views include the LOG view, a raw TEXT view (`t`), a histogram of message volume over time (`i`), and a pretty-print view (`P`).

The whole thing is a C++14 codebase linking PCRE2, SQLite (3.9.0+), zlib, bz2, libcurl, libarchive, and libunistring. `tshark` (Wireshark) is invoked externally to interpret pcap files.

## Production Notes

- **Unrecognized formats degrade silently.** The most common "lnav isn't working" report is a custom application log landing in the TEXT view with no highlighting or SQL columns. There is no error — the lines just aren't treated as logs. Verify recognition early, and budget time to write a format file for in-house log shapes.
- **Timestamps without a year are a trap.** `journalctl`'s default output omits the year; across a year boundary lnav can order messages wrong. Force `-o short-iso` (or the JSON output, which also surfaces `PRIORITY` and `_SYSTEMD_UNIT`). The same caution applies to any terse custom format.
- **Everything is local and in-memory.** The index is rebuilt each run and lives in RAM; there is no persistent index to reuse across invocations. Feeding in an entire long-lived persistent journal or many gigabytes of history means paying indexing cost up front, so constrain input with `journalctl -n` / `--since` / `-b` or by pointing at specific files.
- **Config path moved.** Older docs reference `~/.lnav`; current builds use `~/.config/lnav` (XDG). Custom formats, saved sessions, and keymaps live there. Mixing versions on shared hosts can lead to "my format disappeared" confusion.
- **Building from source is heavier than a typical CLI.** The dependency list is long, and the Rust/`cargo` requirement (for PRQL) surprises people expecting a pure C++ `./configure && make`. For most users the prebuilt static binaries or a package manager are the right call.
- **Single host, single user.** There is no clustering, no auth, no remote query. It is a debugging tool you run next to the logs, not a log platform.

## When to Use / When Not

**Use when:**
- You need to correlate several log files (or a whole directory) in true time order on one machine.
- You want ad-hoc SQL/PRQL analytics over structured logs without standing up a database or log stack.
- You're doing interactive incident debugging: jump between errors, filter with regex, pretty-print JSON, histogram volume spikes.
- You want one static binary with no daemon, index files, or service to manage.

**Avoid when:**
- You need centralized, multi-host log aggregation with retention and alerting — that's Loki/Elastic/Splunk territory.
- Your logs are an unusual in-house format and you're unwilling to write a format definition.
- You only need pure-text search on a single file — `less`/`ripgrep` are lighter.
- You need programmatic, headless log processing in a pipeline rather than an interactive TUI.

## Alternatives

- rcoh/angle-grinder — use instead when you want SumoLogic-style aggregation queries on a stream and don't need a full TUI or multi-file time-merge.
- bensadeh/tailspin — use instead when you just want colorized/highlighted `tail -f` output with zero format configuration.
- variar/klogg — use instead when you want a GUI viewer tuned for searching very large single files.
- jqlang/jq — use instead when your logs are pure JSON and you want scriptable filtering/transformation rather than an interactive viewer.
- grafana/loki — use instead when the real problem is centralized, multi-host log collection, retention, and dashboards rather than single-host debugging.

## History

| Version | Date | Notes |
|---------|------|-------|
| (initial) | 2009 | Repository created; project begins as a terminal log navigator[^1]. |
| 0.8.x | 2015–2017 | SQLite-backed querying and format definitions mature. |
| 0.11.0 | 2023 | PRQL query support added alongside SQL (adds a Rust build dependency). |
| 0.12.x | 2024 | Continued format, UI, and performance work. |
| 0.13.x | 2025–2026 | Ongoing active maintenance; commits through 2026[^2]. |

## References

[^1]: GitHub API metadata for tstack/lnav — repository `created_at` 2009-09-14, license BSD-2-Clause, primary language C++. https://github.com/tstack/lnav
[^2]: GitHub API metadata for tstack/lnav — `pushed_at` in 2026, ~10.4k stars, ~389 forks (fetched 2026-07). https://github.com/tstack/lnav
[^3]: lnav documentation — formats, usage, and SQL/PRQL interface. https://docs.lnav.org

## Tags

c-plus-plus, cli, tui, log-viewer, log-analysis, observability, sqlite, terminal, pager, devops
