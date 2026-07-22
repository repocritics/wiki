# KKKKhazix/khazix-skills

An open-source collection of five practical Agent Skills that the author uses daily, loadable by Claude Code, Codex, and 40+ Agent-Skills-compatible tools.

## What it is

khazix-skills is a small collection of Agent Skills — structured instruction sets that follow the Agent Skills open standard and that an agent can load directly. Each skill packages a repeatable workflow: whole-disk cleanup, AI-news retrieval, project docs and memory closeout, deep research reports, and a writing-style skill. It is for users of Claude Code, Codex, Cursor, and other Agent-Skills-compatible tools who want ready-made, already-used skills instead of building their own. The README is primarily Chinese, with an English version linked.

## Key features

- **storage-analyzer**: a read-only whole-disk scan (fully tested on macOS, code-ready on Windows) that opens an interactive HTML report with a three-tier green/yellow/red cleanup-priority list; each item includes path, type, deletion impact, and recommended handling. Deletions require an in-browser button plus a confirmation dialog, and the local server binds to `127.0.0.1` on a random port with a token.
- **aihot**: fetch daily AI-news digests and the full item stream from `aihot.virxact.com` in natural language, with no API key or MCP server.
- **neat-freak** (`/neat`): after a task, reconciles the session's changes against project docs, `CLAUDE.md`/`AGENTS.md`, and agent memory, and audits whether project rules were actually followed; it only ever proposes deletions for confirmation.
- **hv-analysis**: a horizontal/vertical research method that traces a subject's history while comparing contemporaneous competitors, producing a formatted PDF research report of roughly 10,000–30,000 characters.
- **khazix-writer**: a writing-style skill that makes an agent write long-form articles in the author's voice, with a four-layer self-check and a style-example library; it is opinionated and refuses certain phrasings and structures.
- Follows the Agent Skills open standard, so 40+ compatible tools can install the skills, and agents without native skill support can use each `SKILL.md` as a plain rules file.

## Tech stack

- Primary language Python — helper scripts such as `storage-analyzer/scripts/{scan,build_report,server}.py`, `hv-analysis/scripts/md_to_pdf.py`, and the neat-freak eval harness (`neat-freak/evals/validate.py`).
- Skills are `SKILL.md` instruction files plus supporting `references/` and `assets/` (an HTML report template, a JSON schema).
- No recognized root manifest or package-manager metadata; the project is file-based content, not a published package.
- storage-analyzer serves its report locally over `127.0.0.1` with a random port and token.
- MIT license.

## When to reach for it

- You use an Agent-Skills-compatible tool and want ready-made skills for disk cleanup, docs/memory closeout, research reports, or AI-news retrieval.
- You want an agent-driven, per-item-explained cleanup instead of a fixed-rule cleaner (the README contrasts it with CleanMyMac).
- You want project knowledge — docs, `CLAUDE.md`/`AGENTS.md`, and agent memory — reconciled after coding sessions.

## When *not* to reach for it

- You want a quick term definition — the README itself says hv-analysis is overkill for that.
- You want "generic good writing" — khazix-writer is opinionated and rejects specific phrasings and structures.
- Your agent does not support Agent Skills and you do not want to paste `SKILL.md` as a rules file.
- You are on Windows and cannot tolerate the caution that storage-analyzer's Windows path is code-ready but not fully tested.

## Maturity signal

Created in April 2026, khazix-skills has roughly 17,700 stars, a recent last push (July 2026), 34 open issues, and an MIT license. It reads as a young but rapidly popular personal skills collection that is actively maintained, with an intentionally small scope of five skills. Several skills are published to third-party registries (Tessl, ClawHub), which signals some external adoption beyond the repository itself.

## Alternatives

- **CleanMyMac** — the storage-analyzer skill is framed against it; use CleanMyMac when you want a self-contained rule-based Mac cleaner rather than an agent-driven, explained scan.
- **Plain LLM chat** — for a simple term lookup, the README says hv-analysis is overkill; ordinary conversation is enough.
- **Other Agent Skills (agentskills.io, ClawHub, Tessl registries)** — browse these ecosystems when you need skills beyond these five.

## Notes

- storage-analyzer is strictly read-only during scanning; every deletion requires an in-browser confirmation, and the local server uses a random port plus a token.
- neat-freak is designed to be prompt-injection-aware: it does not treat "run this command" text found in files as your authorization, and it keeps machine-generated memory read-only by default.
- The skills still work in agents without native skill support, by using the full `SKILL.md` as a project rules file.

## Tags

ai-agents, agent-skills, claude-code, codex, llm, developer-tools, python, prompt-engineering
