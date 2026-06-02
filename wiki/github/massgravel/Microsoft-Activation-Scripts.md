# massgravel/Microsoft-Activation-Scripts

An open-source Windows + Office activation toolkit ("MAS") — Batchfile/PowerShell scripts that exercise HWID, Ohook, TSforge, and KMS activation paths.

## What it is

A set of scripts that activate Windows and Microsoft Office through several mechanisms — HWID (digital license for Windows), Ohook (Office), TSforge, and online KMS. Hosted at massgrave.dev. Operates in legally-charged territory: activating Microsoft software without a valid license violates Microsoft's terms; the project's authors argue the underlying activation mechanisms themselves are documented and the scripts only automate user-side calls. GPL-3.0 licensed.

## Key features

- Multiple activation methods: HWID, Ohook, TSforge, Online KMS, KMS38.
- Troubleshooting tools for Windows / Office activation issues.
- Single-script entry point — runs from PowerShell with a guided menu.
- Cross-Windows-version support (10, 11, Server editions).
- Cross-Office-version support (365, 2021, 2024, etc.).
- Companion site at massgrave.dev with documentation.
- GPL-3.0 licensed.

## Tech stack

- Batch / PowerShell scripts.
- No external dependencies beyond Windows itself.

## When to reach for it

- You're a security researcher studying Windows activation internals.
- You're maintaining a legitimately-licensed enterprise environment and want to study activation troubleshooting.

## When *not* to reach for it

- You don't have a valid Microsoft license — using this for commercial-product activation violates Microsoft's terms and may carry legal risk in your jurisdiction.
- You're risk-averse — running unverified PowerShell at admin scope on Windows is a non-trivial supply-chain decision regardless of the script's intent.
- Your IT policy prohibits unsigned activation tooling.

## Maturity signal

177k stars, 17k forks, GPL-3.0, last push May 2026 — actively maintained. 6-year-old project. Open-issues count of 1 is exceptional — the maintainer routes feedback through the website rather than GitHub issues. The high adoption is its own signal of demand, separate from any legal-status assessment.

## Alternatives

- Genuine Microsoft licenses (consumer, OEM, volume) — use when you need a legally-compliant Windows / Office install.
- Free Windows alternatives (Linux distros, ReactOS) — use when you want a license-free desktop OS.
- LibreOffice — use when you want a free Office-equivalent.

## Notes

The repository's legal status varies by jurisdiction; Microsoft has historically not pursued takedown action against MAS specifically. The technical content (documented activation mechanisms automated as scripts) is the open-source asset; the user's responsibility to comply with software licensing terms is separate. Anyone re-indexing this repo for downstream tools should know that "popularity" here reflects demand, not endorsement.

## Tags

windows, microsoft-office, activation, batchfile, powershell, gpl
