# Imbad0202/academic-research-skills

A suite of Claude Code skills that runs the academic paper lifecycle — research, write, review, revise, finalize — as a human-in-the-loop pipeline with integrity gates.

## What it is

Academic Research Skills (ARS) is a set of prompt-driven Claude Code skills covering the full pipeline from research to publication, aimed at researchers who want AI to handle the grunt work (finding references, formatting citations, verifying data, checking logical consistency) while the human keeps ownership of the question, method, interpretation, and argument. It is explicitly positioned as human-in-the-loop rather than fully automated, citing published failure modes of autonomous research systems (implementation bugs, hallucinated results, citation hallucinations, frame-lock) as the reason. It installs into Claude Code as a plugin and produces artifacts such as integrity reports, peer-review reports, and formatted papers.

## Key features

- Four coordinated skills: Deep Research (13-agent team, 8 modes including PRISMA systematic review and a Socratic guided mode), Academic Paper (12-agent writing with Style Calibration, LaTeX hardening, and citation conversion), Academic Paper Reviewer (7-agent multi-perspective peer review with 0–100 rubrics and a Devil's Advocate), and Academic Pipeline (a 10-stage orchestrator).
- Non-skippable integrity gates at Stage 2.5 and Stage 4.5 that run a blocking failure-mode checklist and claim verification; the showcased run reports catching fabricated references and statistical errors before review.
- Citation-faithfulness tooling: locator anchors on every citation and an opt-in claim audit (`ARS_CLAIM_AUDIT=1`) that fetches cited sources and judges whether a claim is actually supported, with HIGH-WARN classes that hard-gate output at the formatter.
- Support for five citation formats (APA 7.0, Chicago, MLA, IEEE, Vancouver) and six paper structures (IMRaD, thematic literature review, theoretical analysis, case study, policy brief, conference paper).
- Optional cross-model verification, model tiering (`ARS_MODEL_TIERING`), Semantic Scholar API verification, and a Material Passport that tracks experiment provenance for externally run experiments.
- Every stage requires a user confirmation checkpoint, and an R&R Traceability Matrix independently verifies author revision claims.

## Tech stack

- Prompt-driven Claude Code skills authored as Markdown `SKILL.md` files, agents, references, and templates; distributed as a Claude Code plugin (`.claude-plugin/marketplace.json`, `plugin.json`).
- Primary language is reported as Python, but the core skills (research / write / review) need no Python — they are prompt-driven; `pyproject.toml` contains only pytest configuration (`pythonpath = ["."]`).
- A real Python interpreter is needed only for optional hardening (a `PreToolUse` write-scope guard, invoked through `bash` via `hooks.json`) and a few opt-in features (revision-patch mode, submission-package verifier, cache/mark-read commands); a missing interpreter degrades cleanly.
- Optional external tools for output formats: Pandoc for DOCX, and tectonic plus Source Han Serif TC for APA 7.0 PDF; Markdown output works without either.
- JSON Schemas are used for contract validation across handoffs and reports.

## When to reach for it

- You are writing or reviewing academic papers inside Claude Code and want structured stages with explicit checkpoints rather than one-shot generation.
- You need citation formatting and verification, including an audit pass that checks whether cited sources actually support the claims.
- You want a multi-perspective peer review with quality rubrics and a decision mapping before submitting or revising.
- You are processing reviewer comments and want a traceable revise-and-resubmit workflow.
- You work in English or Traditional Chinese (or want intent-based Socratic/plan modes that operate in other languages).

## When *not* to reach for it

- You want AI to write the whole paper autonomously; the tool deliberately keeps a human in the loop and will not remove that role.
- You are not using Claude Code (or a compatible import path); the slash commands, hooks, and subagent orchestration do not transfer to other harnesses — a separate Codex distribution exists for that.
- Your use is commercial: the work is licensed CC-BY-NC 4.0 (NonCommercial), which forbids commercial use.
- You need byte-reproducible outputs; the docs state LLM outputs are not byte-reproducible and the reproducibility lockfile is configuration documentation, not a replay guarantee.

## Maturity signal

The project was created in late February 2026 and pushed within days of this writing, with a detailed changelog reaching v3.18.0, so it is actively and rapidly developed. Its large star and fork counts accrued over a few months indicate strong adoption in the Claude Code ecosystem, and the very low open-issue count alongside frequent versioned releases suggests a maintainer keeping pace with reported problems. The license is CC-BY-NC 4.0 (reported by the platform as NOASSERTION), which is a content/creative-works license rather than a permissive code license and constrains commercial reuse. Treat it as actively maintained and mature in feature scope for its niche, with a single primary maintainer.

## Alternatives

- Imbad0202/academic-research-skills-codex — use this sibling distribution instead if you work in Codex CLI rather than Claude Code; it packages the same workflow content Codex-natively.
- Imbad0202/experiment-agent — reach for this companion when your research requires actually running code or human-study experiments before writing, since ARS never runs experiments itself.
- aspi6246/Claude-Code-Skills-for-Academics — use when you want a leaner set of academic Claude Code skills; the README credits it as the inspiration for ARS's read-only constraint and lean-skill philosophy.

## Notes

- The design rationale is grounded in cited literature on autonomous-research failure modes and corpus-scale citation hallucination, and the repo tracks specific survey-driven deltas as issues rather than claiming empirical superiority.
- Several anti-sycophancy mechanisms are explicit: the Devil's Advocate must score each rebuttal 1–5 and may only concede at ≥4, and a Socratic Mentor runs silent dialogue-health self-assessments to detect premature convergence.

## Tags

claude-code, prompt-engineering, academic-writing, literature-review, peer-review, research, llm, multi-agent
