# dair-ai/Prompt-Engineering-Guide

A curated learning guide collecting papers, lessons, notebooks, and references on prompt engineering, context engineering, RAG, and AI agents for large language models.

## What it is

This is an educational resource, not a software library. It gathers guides, papers, lectures, references, and tool listings that explain how to develop and optimize prompts for language models across a range of tasks. The material targets both researchers looking to probe LLM capabilities and limitations, and developers building prompting techniques that interface with LLMs and other tools. The content is published as a web guide at promptingguide.ai and mirrored in this repository as MDX pages.

## Key features

- Structured guide sections covering an introduction, prompting techniques, applications, a prompt hub, model-specific notes, and risks and misuses.
- Technique pages for zero-shot, few-shot, chain-of-thought, self-consistency, generate-knowledge, prompt chaining, tree of thoughts, RAG, ReAct, ART, APE, active-prompt, directional stimulus prompting, PAL, multimodal CoT, and graph prompting.
- Model-specific pages for ChatGPT, GPT-4, Code Llama, Flan, Gemini, LLaMA, Mistral 7B, Mixtral, OLMo, and Phi-2.
- A one-hour video lecture with an accompanying code notebook and slides.
- Multilingual content; the README states 13 languages are supported, and the file tree includes a full Arabic (`ar-pages`) translation set.
- Sections for papers, tools, notebooks, datasets, and additional readings, plus a citation entry for referencing the guide in work or research.

## Tech stack

- Content authored in MDX (the repository's primary language).
- Next.js `^13.5.6` with the Nextra `^2.13.2` docs framework and `nextra-theme-docs` `^2.13.2`.
- React `^18.2.0` and React DOM `^18.2.0`.
- TypeScript `^4.9.5` (dev dependency) for the `.tsx`/`.jsx` UI components.
- KaTeX `^0.16.27` for math rendering, FontAwesome icons, and Vercel Analytics.
- Jupyter notebooks (`.ipynb`) for the lecture and code examples.
- Local development uses Node `>=18.0.0` and `pnpm` (`pnpm i next react react-dom nextra nextra-theme-docs`, then `pnpm dev`).

## When to reach for it

- Learning prompt engineering concepts and terminology from a single organized source.
- Looking up a named technique (for example chain-of-thought, ReAct, or self-consistency) and its associated references.
- Finding papers, datasets, notebooks, and tools related to prompting and LLMs.
- Onboarding a team to prompting fundamentals, including risks such as adversarial prompting, factuality, and biases.

## When *not* to reach for it

- You need an installable library or SDK to run prompts in code; this is documentation, not a runtime.
- You need a framework for building or orchestrating agents; the guide describes techniques rather than providing an implementation.
- You need canonical, provider-specific behavior for a given model; vendor documentation stays more current with frequent model releases.
- You want hands-on, structured coursework; the README points that need to the separate paid DAIR.AI Academy rather than the repository itself.

## Maturity signal

This is a long-running, widely referenced project. It was created in December 2022, reached #1 on Hacker News in February 2023, and reports crossing 3 million learners by January 2024. With roughly 76.8k stars and 8.4k forks, and a last push in March 2026, it is actively maintained and treated as a reference by a large audience. The MIT license and open contribution invitation in the README make reuse and contribution straightforward. As a content repository, "maintenance" means keeping guide pages and model coverage current rather than shipping releases.

## Alternatives

- Learn Prompting (learnprompting.org) — reach for it when you want a more course-like, sequential curriculum instead of a reference-style guide.
- OpenAI Cookbook — reach for it when you want runnable code recipes tied to a specific provider's API rather than technique explanations.
- Anthropic and OpenAI prompt engineering documentation — reach for these when you need provider-specific, up-to-date guidance for a particular model family.
- microsoft/generative-ai-for-beginners — reach for it when you want a broader beginner track across generative AI topics, not prompting alone.

## Notes

- The site is built on the generic `nextra-docs-template`; the root `package.json` still carries the template's name and original author metadata rather than project-specific fields.
- The repository is sponsored by SerpApi (noted in the README header).
- It includes a `CLAUDE.md` and GitHub Actions workflows (`claude-code-review.yml`, `claude.yml`), indicating automated Claude-based code review is wired into the repo.
- The README advertises separate commercial offerings — DAIR.AI Academy courses and corporate training, consulting, and talks — that are distinct from the free guide content.

## Tags

mdx, nextjs, nextra, react, typescript, documentation, prompt-engineering, llm, rag, ai-agents, machine-learning, awesome-list
