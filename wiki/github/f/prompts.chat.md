# f/prompts.chat

A community-curated catalog of LLM prompts (formerly "Awesome ChatGPT Prompts"), distributed as both a Next.js site at prompts.chat and a markdown index in the repo.

## What it is

A collection of role-based and task-specific prompt templates for ChatGPT, Claude, Gemini, and other LLMs. Started in late 2022 as `awesome-chatgpt-prompts`; rebranded to `prompts.chat` as the site grew into a self-hostable web app for sharing, browsing, and copying prompts. The markdown source-of-truth lives in this repo; the site is the polished UX layer; both are deployable / self-hostable.

## Key features

- Hundreds of role-prompts (act as a Linux terminal, English translator, Excel sheet, etc.) plus task-specific templates.
- Multi-model coverage — prompts are framed model-agnostically and tested across ChatGPT, Claude, Gemini.
- Next.js + TypeScript companion site with search, filtering, and copy-to-clipboard.
- Self-hostable for org-specific prompt libraries — privacy-friendly path for teams that don't want a public collection.
- Active contribution flow — PRs against the markdown corpus drive both the README and the site.

## Tech stack

- HTML / TypeScript primary on the site side (Next.js).
- Markdown content as the source-of-truth corpus.
- npm package distribution for the site; markdown is self-rendering on GitHub.

## When to reach for it

- You're prototyping prompt designs and want a starting library of vetted role/task templates.
- You're standing up an internal prompt catalog and want a self-hostable baseline.
- You're studying community prompt-engineering conventions over time — the git history is a primary source.

## When *not* to reach for it

- You want production-graded, parameterized prompt templates with versioning and evals — LangSmith, PromptLayer, or per-app prompt files fit better.
- You want platform-specific prompts (e.g. Claude-Code-specific slash commands) — those live in their own repos.
- You want a license-clean derivative for commercial reuse — SPDX is `NOASSERTION`; verify LICENSE file.

## Maturity signal

163k stars, 21k forks, last push the morning this page was generated — actively maintained. 3-year-old project that survived the late-2022 ChatGPT-prompt-list wave and matured into a sustained reference. Open-issues count of 47 is unusually low for the surface area, signaling effective triage. The rebrand from "awesome ChatGPT prompts" to "prompts.chat" preserves the underlying corpus continuity.

## Alternatives

- LangSmith / PromptLayer — use when you need versioned, evaluated, monitored prompt templates.
- Anthropic prompt library / OpenAI prompt examples — use when you want vendor-curated prompts.
- Custom org wiki — use when prompts are sensitive or use-case-specific.

## Notes

The "prompts.chat" rebrand corresponds to the README's framing as "community + self-host" rather than just a markdown list. License is `NOASSERTION` — anyone reusing the corpus at scale (training data, prompt-recommendation tooling) should pin to the LICENSE file at consumption time. The Next.js site is the supported UX layer; the markdown README is the editable source.

## Tags

awesome-list, prompt-engineering, large-language-model, claude, openai, gemini, typescript, nextjs, self-hosted
