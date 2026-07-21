# VoltAgent/awesome-openclaw-skills

A curated, category-organized awesome-list of thousands of community-built OpenClaw skills drawn from the ClawHub registry.

## What it is

This repository is an awesome-list that discovers, filters, and categorizes community-built OpenClaw skills sourced from ClawHub, OpenClaw's public skills registry. OpenClaw is a locally running AI assistant, and skills extend it to interact with external services, automate workflows, and perform specialized tasks. The list is aimed at OpenClaw users who want to find and install the right skills for their needs, and it also serves as a source of use-case inspiration. Entries are grouped into roughly 29 categories, each with its own detail file.

## Key features

- Thousands of skills organized by category, with per-category detail files under `categories/` (for example Coding Agents & IDEs, Web & Frontend Development, DevOps & Cloud, Browser & Automation).
- A documented filtering pass that excludes 4,065 likely-spam, 1,040 duplicate, 851 low-quality or non-English, 886 crypto/finance, and 373 skills flagged as malicious by published security audits.
- Installation guidance for the OpenClaw CLI (`openclaw skills install`), the ClawHub CLI (`npx clawhub install`), and manual folder placement with a stated priority order.
- An acceptance policy that only lists skills already published on ClawHub, not personal repos or gists.
- A security notice stating that listed skills are curated, not audited, and pointing to VirusTotal reports and third-party scanners.

## Tech stack

- No primary language reported and no recognized root manifests; the repository is Markdown documentation.
- A GitHub Actions workflow (`.github/workflows/pr-check.yml`) for pull-request checks.
- MIT-licensed.

## When to reach for it

- You use OpenClaw and want to browse or discover skills by category rather than searching the raw registry.
- You want a list that has already filtered out bulk spam, duplicates, and entries flagged as malicious.
- You are looking for inspiration on what OpenClaw skills can do.

## When *not* to reach for it

- You need audited or security-guaranteed skills; the list is explicitly curated, not audited, and skills can change after listing.
- You want to include skills that are not published on ClawHub; those are not accepted.
- You need a programmatic API or registry query surface rather than a static Markdown list.
- You are not an OpenClaw user; the list is specific to that ecosystem.

## Maturity signal

The repository is MIT-licensed, actively maintained (last push mid-2026 against an early-2026 creation date), and carries very few open issues, which for an awesome-list suggests steady curation and prompt PR handling rather than a stale collection. The documented filtering counts and a stated contribution policy indicate an actively managed list with editorial process behind it, not an unmaintained dump of links. Treat it as an actively curated reference.

## Alternatives

- ClawHub (clawhub.ai) — use it instead when you want the full, unfiltered registry and per-skill VirusTotal reports rather than a curated subset.
- Other awesome-lists in adjacent ecosystems (for example lists of MCP servers) — use them instead when your agent runtime is not OpenClaw.

## Notes

- The filtering table is unusually specific, breaking out exact exclusion counts by reason, including 373 skills identified as malicious by external security researchers.
- The category index shows how lopsided the ecosystem is, with Coding Agents & IDEs and Web & Frontend Development dominating the skill counts.

## Tags

awesome-list, openclaw, agent-skills, curated-list, documentation, markdown, ai-agents
