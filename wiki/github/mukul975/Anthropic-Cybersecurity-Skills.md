# mukul975/Anthropic-Cybersecurity-Skills

A community-maintained library of 817 structured cybersecurity skills for AI agents, each written to the agentskills.io standard and mapped to six industry security frameworks.

## What it is

This repository is a knowledge base that gives AI agents practitioner-style security workflows they otherwise lack — when to use a technique, what prerequisites to check, how to execute it step by step, and how to verify results. It targets people running security-capable AI agents (analysts, red/blue teams, DFIR and SOC practitioners) who want their agent to follow the same playbook a senior analyst would. Each skill follows the agentskills.io open standard and is mapped across MITRE ATT&CK, NIST CSF 2.0, MITRE ATLAS, MITRE D3FEND, NIST AI RMF, and the MITRE Fight Fraud Framework (F3). It is an independent community project and is not affiliated with Anthropic PBC.

## Key features

- 817 skills spanning 29 security domains, from cloud security and threat hunting to malware analysis, red teaming, and AI security.
- Every skill mapped to six frameworks, including the newer MITRE F3 fraud framework (94 fraud-relevant skills) and MITRE ATT&CK v19.1 (validated with the official `mitreattack-python` library).
- Progressive-disclosure design: roughly 30 tokens to scan a skill's YAML frontmatter and 500–2,000 tokens to load its full workflow, so an agent can search all skills in one pass.
- Consistent skill anatomy: each skill is a directory with `SKILL.md` (YAML frontmatter plus Markdown body), `references/`, `scripts/`, and `assets/`.
- Structured Markdown body sections — When to Use, Prerequisites, Workflow, Verification — that encode real execution steps rather than summaries.
- Loads with zero configuration on any agentskills.io-compatible platform, including Claude Code, GitHub Copilot, Codex CLI, Cursor, and Gemini CLI.

## Tech stack

- Primary language reported as Python; the substance of the repo is structured content — Markdown skill files with YAML frontmatter, plus per-skill Python helper scripts (`scripts/agent.py`, `scripts/process.py`).
- Built to the agentskills.io skill standard, with `.claude-plugin/marketplace.json` and `plugin.json` for plugin distribution.
- Framework mappings maintained as data: `mappings/` includes an ATT&CK Navigator layer, NIST CSF alignment, and OWASP references; F3 mappings live in each skill's `mitre_f3` frontmatter.
- GitHub Actions workflows for skill validation, index generation, and marketplace version sync.
- Apache-2.0 licensed; no packaged root build manifest (installed by cloning or via `npx skills add`).

## When to reach for it

- You want to equip an AI coding or security agent with domain playbooks for investigation, hunting, DFIR, or pentest scoping.
- Your agent platform supports the agentskills.io standard and you want skills that load without configuration.
- You need cross-framework mapping (ATT&CK, CSF, ATLAS, D3FEND, AI RMF, F3) for coverage or compliance conversations.
- You want a broad, consistently structured starting point across many security domains rather than assembling playbooks yourself.

## When *not* to reach for it

- You want ready-to-run tooling or exploit code; this is structured guidance, not an executable toolkit or payload collection.
- Your agent framework does not support the agentskills.io standard, so the frontmatter-driven discovery model does not apply.
- You need authoritative, independently vetted procedures — content is community-contributed and includes offensive/dual-use techniques intended only for authorized, lawful use.
- You expect it to replace a human analyst's judgment rather than augment an agent's.

## Maturity signal

The project has grown quickly, reaching roughly 26k stars within about four months of its February 2026 creation, and the last push is about a month before this snapshot — recently active, though not changing daily. It is Apache-2.0 and not archived, and its open-issue count (around 40) is low relative to its size and star count, which points to an actively curated, community-driven library rather than a neglected one. Content is still expanding on `main` past the tagged v1.0.0 release (734 skills then, 817 now).

## Alternatives

- Use **Atomic Red Team** instead when you want executable, ATT&CK-aligned adversary-emulation tests to run against systems rather than agent-oriented playbooks.
- Use **PayloadsAllTheThings** instead when you want raw payloads, wordlists, and technique snippets rather than structured When/Prerequisites/Workflow/Verification skills.
- Use **MITRE Caldera** instead when you want an automated adversary-emulation platform with its own execution engine.
- Use **VoltAgent/awesome-agent-skills** instead when you want a broad, multi-domain index of agent skills rather than a security-focused library.

## Notes

- Despite the name, the project is explicitly not affiliated with Anthropic; it is the work of an independent maintainer and community.
- It claims to be the only open-source skills library mapping every skill to all six of these frameworks, and it added MITRE's Fight Fraud Framework (F3), released April 2026, verifying all 123 v1.1 technique IDs against the upstream STIX bundle.
- The README carries a prominent authorized-and-lawful-use warning because the library includes offensive and dual-use techniques such as red-team C2, phishing simulation, and exploitation.

## Tags

python, cybersecurity, security, llm, ai-agents, mitre-attack, threat-hunting, incident-response, penetration-testing, knowledge-base, agent-skills
