# x1xhlol/system-prompts-and-models-of-ai-tools

A community-curated collection of leaked or extracted system prompts and internal tool descriptions from popular AI coding tools — Cursor, Windsurf, Devin, Cluely, Lovable, v0, GitHub Copilot, Trae, Replit, and others.

## What it is

A repository that compiles system prompts, internal tool definitions, and (claimed) model configurations of major commercial AI coding products. The provenance varies: some entries come from product documentation, some from prompt-leak / jailbreak techniques, some from reverse-engineered observation. The collection is useful for studying how different products construct their agent prompts but lives in a legally and ethically gray area for vendors whose proprietary prompts are included without authorization.

## Key features

- Wide coverage across the commercial AI coding tool space — Cursor, Windsurf, Devin AI, Cluely, Lovable, v0, GitHub Copilot, Replit, Trae, Manus, NotionAI, Perplexity, Warp.dev, Xcode, Comet, and ~30+ others.
- Per-tool subdirectories with prompt text + (where available) tool descriptions and model configurations.
- Active updates as new prompts get leaked or as products update theirs.
- GPL-3.0 licensed.

## Tech stack

- Markdown / plain-text content only — this is documentation, not code.

## When to reach for it

- You're researching prompt-engineering patterns across commercial agent products and want a side-by-side reference.
- You're building your own AI coding tool and want to study how the established players structure their system prompts.
- You're studying the prompt-leak ecosystem as a research topic.

## When *not* to reach for it

- You want vendor-authorized documentation — many entries are scraped or leaked content. Reuse for commercial purposes carries legal risk.
- You're building a derivative product and plan to clone competitor prompts directly — beyond legal risk, that's a poor differentiation strategy.
- You want vetted, current accuracy — products update prompts continuously; entries here are snapshots that drift.

## Maturity signal

138k stars, 34k forks, GPL-3.0, last push 2026-05-23 — actively maintained. 1+ year old. Open-issues count of 148 is low for the surface area; updates flow as PRs from contributors who've captured new prompts. The high fork count likely reflects users archiving the corpus locally to mitigate possible takedown — DMCA / cease-and-desist actions are a recurring possibility for this kind of collection.

## Alternatives

- Official vendor documentation — use when you want authorized, current, complete reference per product.
- `anthropics/skills`, `obra/superpowers`, `affaan-m/ECC` — use when you want documented agent-skill / system-prompt patterns rather than scraped competitor data.
- `simonw/llm` and similar — use when you want CLI-driven prompt experimentation with your own models.

## Notes

The legal and ethical status of this repository is the most important non-obvious fact. System prompts in commercial products are often considered trade secrets; some products have explicit ToS clauses against extraction. Using this corpus for research / education is generally defensible; cloning competitor prompts verbatim into a derivative product is not. GPL-3.0 doesn't override the underlying IP claims of the original prompt authors. Anyone consuming this corpus at scale (training data, derived products) should consult legal counsel.

## Tags

artificial-intelligence, large-language-model, system-prompts, agent, awesome-list, claude, openai, cursor, copilot, gpl
