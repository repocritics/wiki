# github/spec-kit

GitHub's official toolkit for "Spec-Driven Development" — a workflow + tooling combo that puts product specifications at the center of AI-assisted engineering.

## What it is

A Python-based toolkit from GitHub that codifies the "Spec-Driven Development" methodology: write a structured product specification, derive an implementation plan, then have AI tools execute against both. The kit provides templates, commands, and integration points with GitHub Copilot and other coding agents. Aims to bring product-management rigor (PRDs, acceptance criteria, traceability) into the AI-engineering loop.

## Key features

- Structured spec templates (PRD-style) with traceability between intent → spec → plan → code.
- CLI commands for generating, validating, and tracking specs.
- Tight integration with GitHub Copilot's agent surface.
- Python-implemented; pip-installable.
- MIT-licensed.

## Tech stack

- Python primary.
- CLI distributed via pip / pipx.
- Templates as markdown files with structured metadata.

## When to reach for it

- You're using AI coding agents heavily and want a structured methodology to channel them.
- You're a tech-lead trying to bridge product specs and AI-generated code with traceability.
- You're evaluating "spec-first" workflows for team-wide adoption.

## When *not* to reach for it

- You're shipping prototypes where structure-overhead doesn't pay off.
- Your team has a different established workflow (PRDs in Notion, tickets in Linear, etc.) and wants to keep the spec format there.
- You want to avoid GitHub-specific tooling lock-in — the integration depth with Copilot is part of the value, and part of the lock-in.

## Maturity signal

107k stars in ~10 months (August 2025 origin) under github org — fast-rising but proportional given the official GitHub stewardship. Last push the day this page was generated. The 428 open-issues count tracks the breadth of integration requests and workflow questions. GitHub's MIT license + official stewardship signal sustained investment.

## Alternatives

- Custom AGENTS.md / CLAUDE.md files per-project — use when you want zero framework, just project conventions.
- Linear / Notion docs + manual prompts — use when you don't want spec-as-code.
- `obra/superpowers`, `affaan-m/ECC` — third-party agent-skills frameworks, partial overlap.

## Notes

"Spec-Driven Development" as a branded methodology has GitHub as its primary advocate; the kit is the operational form. The official-from-GitHub framing differentiates this from third-party agent-skills experiments — for orgs already on GitHub Copilot, this is the path of least friction. License (MIT) keeps the tooling portable; the methodology branding is harder to use without the supporting tooling.

## Tags

python, awesome-list, developer-tools, agent, spec, github-copilot, methodology
