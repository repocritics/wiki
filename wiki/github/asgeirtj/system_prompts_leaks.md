# asgeirtj/system_prompts_leaks

A regularly updated, vendor-organized collection of system prompts captured verbatim from commercial AI assistants and coding tools.

## What it is

This repository gathers the hidden system prompts that products like ChatGPT, Claude, Gemini, Grok, Perplexity, Copilot, Cursor and many others receive before a user's first message. Prompts are stored as Markdown files organized by vendor and product, and the repo is refreshed as new models and captures appear. It is aimed at people studying prompt engineering, researchers, and anyone curious about how these assistants are configured under the hood.

## Key features

- Verbatim system-prompt captures across many vendors, including Anthropic, OpenAI, Google, xAI, Perplexity, Microsoft, Cursor, Meta, Mistral, Moonshot (Kimi) and DeepSeek.
- Deep coverage of Anthropic's Claude Code, including subagents, bundled skills, slash commands, injected reminders, and MCP server prompts.
- A "Recently Updated" table tracking the latest additions with dates and links.
- Separate sections for officially published prompts, raw captures, and older model versions.
- Content released under CC0-1.0, placing it in the public domain.

## Tech stack

- Content is authored as Markdown files organized into per-vendor directories.
- GitHub reports the primary language as JavaScript, but no root build manifests were found in the inputs, so there is no packaged library or CLI to install.
- Distribution is the Git repository itself; consumption is by reading or cloning the files.

## When to reach for it

- Studying how commercial assistants are instructed, for prompt-engineering research.
- Comparing how different vendors and model versions structure their system prompts.
- Journalism or analysis that needs source material on assistant behavior (the README notes The Washington Post and CEPS' AI World built pieces from it).
- Pulling reference material for Claude Code tooling such as subagents, skills, and injected reminders.

## When *not* to reach for it

- You need runnable code or a library — this is a documentation collection, not software.
- You need an authoritative or official specification; these are captured or leaked artifacts, not vendor-published contracts.
- You need a prompt guaranteed to match a specific deployed model version, since prompts change and captures may lag or be superseded.

## Maturity signal

With roughly 59k stars, a last push on the same day these facts were collected, no archived flag, and a CC0-1.0 license, this is an actively and frequently maintained collection rather than a one-off dump. The steady stream of dated updates and the breadth of vendors covered indicate ongoing curation, and the permissive public-domain license makes reuse straightforward.

## Alternatives

- **jujumilk3/leaked-system-prompts** — use it when you want a comparable community-maintained archive of leaked system prompts to cross-reference captures.
- **x1xhlol/system-prompts-and-models-of-ai-tools** — use it when your focus is specifically AI coding tools and their prompts rather than general chat assistants.
- **LouisShark/chatgpt_system_prompt** — use it when you mainly want ChatGPT and custom-GPT instructions rather than a multi-vendor spread.

## Notes

The repository extends well beyond chat models: it captures product integrations such as Claude Design, Claude Cowork, Claude for Microsoft 365, Codex, and Copilot variants, plus runtime-injected "reminder" instructions that are inserted mid-conversation rather than at the system-prompt layer.

## Tags

markdown, awesome-list, llm, prompt-engineering, system-prompts, ai-agents, documentation, reference, chatbot, dataset
