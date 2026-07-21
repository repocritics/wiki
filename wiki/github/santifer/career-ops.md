# santifer/career-ops

An open-source job-search command center that turns any AI coding CLI into a pipeline to scan portals, score listings A-F, tailor CVs, and track applications locally.

## What it is

career-ops runs inside an AI coding CLI (Claude Code, Codex, Gemini/Antigravity, OpenCode, Qwen, Grok, and others) and turns job hunting into a structured pipeline. It evaluates a job description against your CV, produces an A-F score, generates ATS-tailored PDFs, drafts outreach, and tracks everything in one place, while you keep the final decision. It is positioned as a filter for finding the few listings worth your time, explicitly not a mass auto-applier.

## Key features

- Auto-pipeline: paste a URL or JD and get an evaluation, a tailored PDF, and a tracker entry.
- A six-block structured evaluation (role summary, CV match, level strategy, comp research, personalization, interview prep) plus a Block G posting-legitimacy check that flags scams and ghost jobs.
- Portal scanner with 45+ preconfigured companies and 21 provider modules across Greenhouse, Ashby, Lever, and Wellfound; `--verify` runs a Playwright liveness check to drop expired postings.
- ATS-optimized CV and cover-letter PDF generation from HTML templates via Playwright, plus draft-only application emails.
- A Go/Bubble Tea terminal dashboard to browse, filter, and sort the pipeline.
- Human-in-the-loop by design: the system drafts and recommends but never submits, sends, or clicks anything.
- CLI-agnostic through the Agent Skill Standard, with a SKILL.md referenced per supported CLI.

## Tech stack

- JavaScript/Node.js, distributed as `@santifer/career-ops` (package version 1.22.0), run via `npx`.
- `playwright` 1.61.1 for browser automation and PDF rendering; `@google/generative-ai` `^0.24.1`, `dotenv` `^17.0.0`, `js-yaml` `^4.1.1`.
- A dashboard TUI written in Go with Bubble Tea and Lipgloss (Catppuccin Mocha theme).
- Data stored as Markdown tables, YAML config, and TSV batch files.
- Scanner combines Playwright, the Greenhouse API, and WebSearch; batch processing uses headless CLI workers such as `claude -p` or `opencode run`.

## When to reach for it

- Running a focused, filter-first job search from an AI CLI you already use.
- Generating per-listing ATS-tailored CVs and cover letters.
- Tracking a pipeline of applications with automated merge, dedup, status normalization, and health checks.
- Batch-evaluating many listings in parallel with headless workers.

## When *not* to reach for it

- You want spray-and-pray mass applying; the system recommends against applying to anything scoring below 4.0/5.
- You expect fully automated submission; by design it never submits, sends, or clicks.
- You have no AI coding CLI or no Node.js available.
- You want a hosted, zero-setup service; this is a local tool where you supply your own model and keys.

## Maturity signal

With roughly 61k stars, creation in April 2026, a push on the same day these facts were gathered, an MIT license at version 1.22.0, and around 200 open issues, this is a very young project growing and iterating rapidly. The large number of `.mjs` utilities and the high open-issue count point to fast, active development; the author notes the repo is maintained in about four hours a week with help from a fleet of AI agents.

## Alternatives

- **feder-cr/Auto_Jobs_Applier_AI_Agent (AIHawk)** — use it when you want full automated applying, which is the opposite of career-ops's human-in-the-loop, filter-first philosophy.
- **Reactive Resume** — use it when you only need an open-source resume builder without portal scanning, evaluation, or tracking.

## Notes

The core design principle is that career-ops is a filter, not an auto-applier: it drafts, scores, and ranks, but the human always submits. It can run entirely on free or local models (OpenRouter free models, Ollama, or any OpenAI-compatible endpoint), and even without installing a CLI via a standalone Gemini API script on the free tier. It is described as the first reference implementation of the CareerOps Manifesto, where signing the manifesto becomes a commit.

## Tags

javascript, nodejs, cli, ai-agent, job-search, automation, playwright, golang, resume, llm
