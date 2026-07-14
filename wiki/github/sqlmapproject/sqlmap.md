# sqlmapproject/sqlmap

> Command-line tool that automates detecting and exploiting SQL injection flaws, then escalating from a single injectable parameter to full database — and often host — takeover.

[GitHub repo](https://github.com/sqlmapproject/sqlmap) ·
[Official website](https://sqlmap.org) ·
[License: GPL-2.0](https://github.com/sqlmapproject/sqlmap/blob/master/LICENSE)

## Overview

sqlmap is an offensive-security tool that finds SQL injection points in a web application and then weaponizes them. Given a URL, request file, or crawl target, it fingerprints the backend DBMS, enumerates databases/tables/columns, dumps data, reads and writes files on the database server, and — where the DBMS and privileges allow — opens an interactive OS shell on the host. It has been the default SQL-injection tool in penetration testing since the late 2000s and remains actively developed, with tagged releases still shipping in 2026[^1].

The project began in 2006 as a student effort by Daniele Bellucci; primary development was taken over by Bernardo Damele A. G. and Miroslav Štampar, who have driven it for most of its life[^2]. It is written in pure Python with no third-party runtime dependencies — a deliberate choice that keeps it runnable on any box with a stock interpreter, at the cost of reimplementing a lot of HTTP and encoding machinery in-tree. It still supports Python 2.7 alongside 3.x, which is unusual for a project maintained into 2026 and reflects a "runs everywhere a pentester lands" priority over modern-runtime hygiene.

The defining tension: sqlmap is enormously capable and correspondingly dangerous. The same switches that dump a table can, at higher `--risk`/`--level`, issue destructive stacked queries or trip data-loss conditions on the target. It is a tool that assumes you have written authorization to attack the system in front of you, and it does very little to stop you if you don't.

## Getting Started

```bash
git clone --depth 1 https://github.com/sqlmapproject/sqlmap.git sqlmap-dev
cd sqlmap-dev
python sqlmap.py -h    # basic options; -hh for the full list
```

There is nothing to install and no dependency tree — `sqlmap.py` runs against the standard library. A minimal authorized test against a single GET parameter:

```bash
# Detect injection, fingerprint DBMS, list databases — non-interactive
python sqlmap.py -u "https://target.example/item?id=1" \
    --batch --dbms=mysql --dbs

# Escalate: dump a specific table
python sqlmap.py -u "https://target.example/item?id=1" \
    --batch -D shop -T users --dump
```

`--batch` accepts every default prompt (use it in scripts). For authenticated or complex requests, save a raw HTTP request from your proxy and pass it with `-r request.txt`. `--wizard` walks beginners through option selection.

## Architecture / How It Works

sqlmap is a single-process CLI orchestrating a fixed pipeline: request handling → injection detection → technique confirmation → enumeration/exploitation. The moving parts worth understanding:

- **Payload catalog is data, not code.** Detection and exploitation payloads live in XML under `data/xml/` (boundaries and per-technique payload templates). Adding or tuning a test is an XML edit, not a code change, which is how the tool covers so many DBMS/technique combinations.
- **Five-plus injection techniques**, selectable via `--technique=BEUSTQ`: **B**oolean-based blind, **E**rror-based, **U**NION query, **S**tacked queries, **T**ime-based blind, and inline **Q**ueries — plus out-of-band data retrieval over DNS exfiltration when a resolver path exists.
- **`--level` (1–5) and `--risk` (1–3)** control breadth and aggressiveness. Higher levels test more parameters (headers, cookies, User-Agent); higher risk enables payloads that can mutate data (e.g. OR-based tests that may match every row in an `UPDATE`).
- **Broad DBMS backends.** MySQL/MariaDB, PostgreSQL, Oracle, Microsoft SQL Server, SQLite, IBM DB2, Access, Firebird and dozens more, each with its own fingerprint signatures and enumeration SQL[^3].
- **Takeover layer.** `--file-read`/`--file-write` for filesystem access, `--sql-shell` for interactive queries, and `--os-shell`/`--os-pwn` for OS command execution (the latter can stage a Meterpreter session). These depend on DBA privileges and DBMS-specific primitives (e.g. UDF injection, `xp_cmdshell`, `INTO OUTFILE`).
- **WAF/IPS evasion via tamper scripts** (`--tamper=`), Python transforms applied to each payload (case swapping, comment injection, encoding) under `tamper/`.
- **Automation surface.** `sqlmapapi.py` exposes a REST/JSON server so other tools can drive scans programmatically.

## Production Notes

sqlmap is an operator's tool, not a service; the caveats are about safe and effective use rather than uptime.

- **`--risk=3` can damage data.** It enables OR-based and stacked-query payloads. On endpoints backed by `UPDATE`/`DELETE` statements these can affect far more rows than intended. Do not raise risk on production-like targets without understanding the query behind the parameter.
- **It is loud.** A real run issues hundreds to thousands of requests, and blind (especially time-based) extraction is slow and generates obvious traffic patterns. Expect WAFs and rate limiters to flag it; tamper scripts help but are an arms race, not a cloak. Tune with `--threads`, `--delay`, and `--time-sec`.
- **False positives happen.** Aggressive detection on noisy endpoints can misreport injection; confirm findings and narrow with `--technique` and `--dbms` when you know the stack.
- **OS takeover has hard preconditions.** `--os-shell`/`--os-pwn` need the right privileges and DBMS support (stacked queries, writable paths, UDF capability). When those aren't present the switches simply fail — the limitation is the target, not the tool.
- **`sqlmapapi.py` has no authentication by default.** Do not expose the API server on an untrusted network; anyone who can reach it can launch scans from your host.
- **Session resumption is stateful.** sqlmap caches results in an output directory keyed by target; a re-run may reuse prior findings. Use `--flush-session` when the target changed or a run went wrong.
- **Legal exposure is the real operational risk.** Running sqlmap against a system you are not authorized to test is a crime in most jurisdictions. Scope authorization in writing before pointing it anywhere.

## When to Use / When Not

**Use when:**
- You have authorization and need to confirm, characterize, and exploit a suspected SQL injection (pentest, bug bounty, CTF).
- You need DBMS fingerprinting and proof-of-impact data extraction quickly across many database backends.
- You want to demonstrate escalation from injection to file/OS access as part of a report.

**Avoid when:**
- You need general web-vulnerability coverage — sqlmap only does SQL injection. Use a full DAST scanner.
- You want continuous, low-noise scanning integrated into CI — it is an interactive attack tool, not a monitoring pipeline.
- You lack written authorization for the target. There is no legitimate "just testing" use against systems you don't control.
- The backend is a NoSQL store — sqlmap targets SQL DBMSs; use a NoSQL-specific tool.

## Alternatives

- r0oth3x49/ghauri — modern SQLi tool positioned as faster with fewer false positives; use when sqlmap's noise or detection tuning frustrates you.
- portswigger/burp — Burp Suite covers the whole web-app attack surface (SQLi included); use when you need a broad proxy-driven scanner rather than one injection specialist.
- zaproxy/zaproxy — OWASP ZAP, open-source DAST for general web scanning; use when you want free, broad coverage over deep SQLi exploitation.
- commixproject/commix — command-injection analog with similar CLI ergonomics; use when the flaw is OS command injection, not SQL.
- codingo/NoSQLMap — use when the backend is MongoDB or another NoSQL store sqlmap can't touch.

## History

| Version | Date | Notes |
|---------|------|-------|
| 0.x (start) | 2006 | Project started by Daniele Bellucci as a student effort[^2]. |
| 0.x (core dev) | ~2009 | Bernardo Damele A. G. and Miroslav Štampar take over primary development[^2]. |
| 0.9 | 2011 | Rewritten detection engine; expanded technique/DBMS coverage[^1]. |
| 1.0 | 2016 | First 1.0 release; move to rolling monthly point releases[^1]. |
| 1.10.7 | 2026-07 | Current tagged release; active development continuing[^1]. |

## References

[^1]: sqlmap releases and changelog. https://github.com/sqlmapproject/sqlmap/releases — repository metadata (37.9k stars, 6.3k forks, last push 2026-07-13) via GitHub API.
[^2]: Project overview and authorship, sqlmap official site and README. https://sqlmap.org
[^3]: sqlmap Features / User's Manual (supported DBMS and techniques). https://github.com/sqlmapproject/sqlmap/wiki/Usage

## Tags

python, security, sql-injection, penetration-testing, appsec, database, offensive-security, exploitation, cli, pentesting, web-security, dbms
