# tconbeer/harlequin

> A full-screen SQL IDE that runs entirely inside your terminal, built on the Textual TUI framework and DuckDB-first by default.

[GitHub repo](https://github.com/tconbeer/harlequin) ·
[Official website](https://harlequin.sh) ·
[License: MIT](https://github.com/tconbeer/harlequin/blob/main/LICENSE)

## Overview

Harlequin is a terminal user interface (TUI) database client written in Python
and built on Textual, the terminal-app framework from Textualize[^1]. It gives
you a persistent full-screen layout — query editor, a browsable data catalog,
and a results grid — rather than the line-by-line REPL that most SQL shells
offer. It is developed primarily by Ted Conbeer, the author of the `sqlfmt`
formatter, and first appeared in 2023 focused exclusively on DuckDB[^2].

The project's defining decision was to generalize away from that DuckDB origin.
Database connectivity is now provided by *adapters* — separate Python packages
discovered through entry points — while the DuckDB and SQLite adapters ship in
the box[^3]. This keeps the core app database-agnostic and lets the community
add Postgres, MySQL/MariaDB, ODBC, BigQuery, and others without touching
Harlequin itself. The tradeoff is uneven: the first-party adapters are well
tested, but adapter quality and feature depth vary across the ecosystem, and a
single connection string can behave differently depending on which plug-in
serves it.

The other defining tension is the medium. A TUI SQL IDE is genuinely useful over
SSH and in keyboard-only workflows where a browser-based tool (DBeaver, DataGrip)
is unavailable, but it inherits every constraint of the terminal: rendering
fidelity depends on the emulator, responsiveness depends on connection latency,
and very large result sets stress both memory and the display layer.

## Getting Started

The maintainers recommend installing into an isolated environment so adapter
dependencies do not collide with your project's. `uv tool` (or `pipx`) does this
automatically:

```bash
uv tool install harlequin
# or: pipx install harlequin
# or, plain: pip install harlequin   (Python 3.9+)
```

Adapters that ship as extras install alongside the app:

```bash
uv tool install 'harlequin[postgres,mysql,s3]'
```

Running it — DuckDB is the default adapter, so no arguments opens an in-memory
DuckDB session; a path opens (or creates) a database file:

```bash
harlequin                          # in-memory DuckDB
harlequin "path/to/duck.db"        # open/create a DuckDB file
harlequin -a sqlite "app.db"       # SQLite adapter
harlequin -a postgres "postgresql://user@localhost:5432/mydb"
```

Inside the app, write a query in the editor and press `Ctrl+Enter` (or `F9`) to
run it; `F1` shows the in-app help. Run `harlequin --help` to see options
contributed by every installed adapter.

## Architecture / How It Works

Harlequin is a Textual application. Textual renders a widget tree to the terminal
using Rich, runs an async event loop, and repaints on state changes[^1]. The
main screen composes three regions: a code editor (with SQL syntax highlighting
and autocomplete), a tree-shaped **data catalog** that introspects the connected
database's schemas/tables/columns, and a **results viewer** backed by a
virtualized data table so it can page through rows without instantiating every
cell widget.

Connectivity is fully decoupled from the UI through the adapter interface. Each
adapter is a Python package that implements Harlequin's connection/cursor
abstraction and registers itself via a setuptools/`importlib.metadata` entry
point under the `harlequin.adapters` group[^3]. On startup Harlequin enumerates
installed adapters, selects one by the `--adapter`/`-a` flag (default `duckdb`),
and asks it to build a connection from the CLI options and connection strings.
Adapters can declare their own CLI options, which is why `--help` output changes
with what you have installed. DuckDB and SQLite adapters are vendored in the main
package; everything else is a distinct dependency.

Query execution is synchronous at the driver level but wrapped so the UI thread
stays responsive: a running query can be cancelled, and results are pulled into
an in-memory buffer that feeds the results table and the export routines.
Configuration (themes, keymaps, profiles, locale) is layered from CLI flags over
a TOML config file, so a named profile can pin an adapter, connection string, and
appearance in one place[^4].

## Production Notes

- **DuckDB storage-format coupling is the most common footgun.** Harlequin pins
  a specific DuckDB version through its bundled adapter, and DuckDB's on-disk
  format is not always forward/backward compatible across releases. A `.db` file
  written by a different DuckDB version than Harlequin's can refuse to open; the
  project maintains a dedicated troubleshooting page for reconciling the
  versions[^5]. If you use DuckDB elsewhere in a pipeline, keep the versions in
  step.
- **Large result sets live in memory.** Results are buffered to drive the table
  and exports, so a `SELECT *` against a huge table can spike memory and make the
  grid sluggish. Add `LIMIT`, or push heavy aggregation into SQL rather than
  scrolling raw rows.
- **Latency shows.** Textual repaints on interaction; over a high-latency SSH
  link keystrokes and scrolling feel laggy in a way a plain readline shell does
  not. It is still usable remotely — that is a core use case — but expect the UI
  to feel heavier than `psql`.
- **Terminal capability matters.** Truecolor and a good Nerd/Unicode-capable font
  give the intended appearance; older or minimal terminals degrade colors and box
  drawing. Copy/paste and some key bindings depend on terminal support and are a
  frequent troubleshooting topic[^5].
- **Install in isolation.** Because adapters pull real database drivers (libpq,
  mysqlclient, ODBC managers), installing Harlequin into a shared/global
  environment invites dependency conflicts. `uv tool`/`pipx` isolation is the
  supported path; the community Homebrew formula bundles several adapters and is
  explicitly noted as less rigorously tested than the PyPI installs[^2].

## When to Use / When Not

**Use when:**
- You want an IDE-like SQL experience (catalog + editor + grid) without leaving
  the terminal or over SSH.
- You work primarily with DuckDB or SQLite and want a batteries-included client.
- You prefer keyboard-driven, mouse-optional tooling and a TOML-configurable,
  themeable setup.

**Avoid when:**
- You need a heavy graphical client with ER diagrams, visual query builders, and
  rich charting — DBeaver or DataGrip fit better.
- Your workflow is scripted/non-interactive; a plain CLI (`duckdb`, `psql`) or a
  library is more appropriate than a full-screen app.
- Your target database only has a community adapter whose maturity you have not
  verified.

## Alternatives

- dbcli/pgcli — use instead when you want a lightweight Postgres REPL with
  autocompletion and no full-screen UI.
- dbcli/litecli — use instead for a focused SQLite command-line client.
- danvergara/dblab — use instead when you want a Go TUI database client covering
  several SQL engines without a Python install.
- duckdb/duckdb — use the official DuckDB CLI instead when you only need DuckDB
  and prefer the canonical shell with matching storage format.
- harelba/q — use instead when you just want to run SQL against CSV/TSV files
  ad hoc rather than manage database connections.

## History

| Version | Date | Notes |
|---------|------|-------|
| initial | 2023-05 | Repo created; DuckDB-only TUI SQL IDE on Textual[^2]. |
| 1.x | 2024 | Adapter architecture generalized connectivity beyond DuckDB; SQLite added in-box, third-party adapters (Postgres, MySQL, ODBC, etc.) via plug-ins[^3]. |
| ongoing | 2026 | Actively maintained by Ted Conbeer; ~6.3k stars, adapter ecosystem and docs at harlequin.sh continue to grow[^6]. |

## References

[^1]: Textual — Python TUI framework by Textualize (Rich-based rendering, async event loop). https://textual.textualize.io/
[^2]: Harlequin README and install guidance (uv/pipx isolation, community Homebrew formula). https://github.com/tconbeer/harlequin
[^3]: Harlequin adapters documentation — adapters as separate packages discovered via entry points. https://harlequin.sh/docs/adapters
[^4]: Harlequin config-file documentation (TOML profiles: adapter, connection, theme, keymaps). https://harlequin.sh/docs/config-file/index
[^5]: Harlequin troubleshooting — DuckDB version mismatch, key bindings, appearance, copy/paste. https://harlequin.sh/docs/troubleshooting/index
[^6]: GitHub repository metadata, retrieved 2026-07-17. https://github.com/tconbeer/harlequin

## Tags

python, sql, terminal, tui, database, duckdb, sqlite, textual, cli, developer-tools, database-client
