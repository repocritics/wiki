# dbcli/pgcli

> A terminal Postgres client that adds autocompletion and syntax highlighting to the `psql` workflow.

[GitHub repo](https://github.com/dbcli/pgcli) ·
[Official website](https://pgcli.com) ·
[License: BSD-3-Clause](https://github.com/dbcli/pgcli/blob/main/LICENSE.txt)

## Overview

pgcli is a command-line interface for PostgreSQL, first released in 2014 by Amjith Ramanujam[^1]. It is a drop-in alternative to the stock `psql` REPL, adding context-aware autocompletion (table and column names pulled from the live schema), Pygments syntax highlighting, multi-line editing, and a config file. The project is the flagship of the broader **dbcli** family, which applies the same interface to other engines — mycli (MySQL), litecli (SQLite), mssql-cli, and iredis among them. As of 2026 it sits around 13.3k stars with steady, low-churn maintenance[^2].

The defining tension of pgcli is that it is a nicer front-end, not a replacement engine. It speaks to Postgres through the same libpq-based driver stack `psql` uses, honors the same `PG*` environment variables and connection URIs, and defers to the server for everything that matters. What it adds is interactive ergonomics. That makes it excellent for humans exploring a schema by hand and a poor fit for anything scripted or non-interactive, where `psql -c` or a real client library remain the correct tools. The autocompletion — its headline feature — depends on introspecting `information_schema` and system catalogs, which means it can be slow or noisy against very large schemas and adds a second connection by default.

## Getting Started

```bash
pip install -U pgcli
# or: brew install pgcli        (macOS)
# or: sudo apt-get install pgcli (Debian/Ubuntu)
# or: uvx pgcli                  (run without installing)
```

```bash
# Connect by database name, by URI, or via PG* env vars (like psql)
$ pgcli local_database
$ pgcli postgresql://user:pass@example.com:5432/app_db?sslmode=verify-ca

# Inside the REPL: autocompletion is on as you type
user@host:app_db> SELECT * FROM <TAB>          -- suggests table names
user@host:app_db> SELECT id FROM users WHERE <TAB>   -- suggests columns
```

A config file is created at `~/.config/pgcli/config` on first launch; it documents every option inline (keyword-casing, row limits, named-query aliases, DSN aliases, key bindings).

## Architecture / How It Works

pgcli is a Python application built on a small stack of well-chosen libraries rather than a monolith:

- **prompt_toolkit** — Jonathan Slenders' pure-Python readline replacement provides the entire interactive layer: the multi-line buffer, key bindings, the completion menu, and history[^3]. pgcli is effectively a prompt_toolkit application with SQL smarts bolted on, and the project's fortunes track prompt_toolkit's closely.
- **psycopg** — the database driver. pgcli migrated from psycopg2 to psycopg 3 in its 4.x line; this is the actual Postgres connection and where real query execution happens.
- **Pygments** — SQL lexer used for syntax highlighting as you type.
- **Click** — command-line argument parsing and the `pgcli --help` surface.
- **pgspecial** — a shared dbcli library implementing the backslash meta-commands (`\d`, `\dt`, `\l`, etc.) that mimic a subset of `psql`.

The autocompletion engine is the non-trivial part. On connect, pgcli queries the catalogs to build an in-memory index of schemas, tables, columns, functions, and keywords, then a completion refresher keeps it current. Suggestions are context-sensitive: the SQL statement is parsed (via sqlparse) far enough to know whether the cursor is in a `FROM`, a `WHERE`, or a function-argument position, and the candidate set is filtered accordingly ("smart completion"). By default this metadata work runs on a **separate connection** so it does not block your foreground query; `--single-connection` disables that at the cost of interactivity.

The dbcli projects deliberately share code: prompt_toolkit, pgspecial/mssqlcli-equivalents, cli_helpers (table formatting), and configobj-based config are common across pgcli, mycli, and litecli. A change to the shared formatting or special-command layer lands in all of them.

## Production Notes

- **This is an interactive tool, not automation.** There is no stable, scriptable output contract. For cron jobs, migrations, or CI use `psql` (`-c`, `-f`, `--csv`, `-tA`) or a language client. Piping into pgcli is not the supported path.
- **Large schemas cost you.** Completion metadata is fetched by introspection; on databases with tens of thousands of tables/columns the initial and periodic refresh can be slow and memory-heavy. Narrow the search path or accept the latency. The second background connection also doubles your connection count per session, which matters against connection-limited or PgBouncer-fronted servers.
- **PgBouncer / transaction pooling.** Because pgcli opens a second connection and relies on session-level introspection, it interacts awkwardly with transaction-mode poolers. `--single-connection` helps.
- **Destructive-query guard.** `--warn` (`all|moderate|off`) prompts before `DROP`/`DELETE`/`UPDATE`-style statements; useful but off-by-context, so do not treat it as a safety net.
- **Password handling.** pgcli reads `~/.pgpass` and the `PG*` variables like `psql`. Passwords passed inline in a connection URI can leak into shell history — prefer `.pgpass`, `PGPASSWORD`, or the interactive prompt.
- **Python version churn.** pgcli tracks modern Python aggressively: it dropped Python < 3.8 at 4.0.0 and continued raising the floor in the 4.x series. Pin your interpreter or install via `pipx`/`uvx` to isolate it from system Python.
- **Output paging.** Wide results are sent through a pager (`pspg` is a popular companion). Auto-vertical output (`\x auto`) reformats rows that exceed terminal width, which is the single most useful setting for wide tables.

## When to Use / When Not

**Use when:**
- You explore Postgres schemas interactively and want tab-completion of real table/column names.
- You want syntax highlighting, multi-line editing, and a saner history than `psql` out of the box.
- You work across engines and want the same ergonomics (pair with mycli/litecli).

**Avoid when:**
- You need scripted, machine-readable, or reproducible output — use `psql` or a client library.
- You run against a massive schema where introspection latency outweighs the completion benefit.
- You are constrained on connections (transaction-pooled PgBouncer) and cannot afford the second connection.

## Alternatives

- postgres/psql — the official reference client; scriptable, ubiquitous, no completion frills. Use when you need automation or guaranteed availability.
- dbcli/mycli — the same tool for MySQL/MariaDB; use when your engine is MySQL, not Postgres.
- dbcli/litecli — the SQLite sibling from the same family.
- dbeaver/dbeaver — heavyweight GUI database client; use when you want visual schema browsing, ER diagrams, and multi-engine management over a terminal.
- tconbeer/harlequin — a modern TUI SQL IDE (Textual-based) with a Postgres adapter; use when you want a full-screen terminal UI rather than a REPL.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2014-10 | First public release; prompt_toolkit-based Postgres REPL by Amjith Ramanujam[^1]. |
| 1.x–2.x | 2016–2019 | Smart completion, named queries, pgspecial backslash commands matured. |
| 3.0 | 2020 | prompt_toolkit 3 migration; Python 2 support removed. |
| 4.0.0 | — | Dropped Python < 3.8; psycopg (v3) driver line. |
| 4.2.0 | — | Dropped Python < 3.9. |
| 4.5.0 | — | Dropped Python < 3.10; continued 4.x maintenance. |

Exact release dates for the 4.x point releases vary; consult the project changelog for authoritative values[^4].

## References

[^1]: pgcli project home and history. https://pgcli.com
[^2]: GitHub repository metadata (dbcli/pgcli): ~13.3k stars, ~602 forks, BSD-3-Clause, last pushed 2026-06-03. https://github.com/dbcli/pgcli
[^3]: Python Prompt Toolkit by Jonathan Slenders — the interactive backbone of pgcli. https://github.com/prompt-toolkit/python-prompt-toolkit
[^4]: pgcli changelog. https://github.com/dbcli/pgcli/blob/main/changelog.rst

## Tags

python, postgresql, database, cli, repl, terminal, autocompletion, syntax-highlighting, developer-tools, dbcli, sql
