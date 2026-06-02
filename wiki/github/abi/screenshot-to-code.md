# abi/screenshot-to-code

A TypeScript tool that converts UI screenshots into clean front-end code — HTML/Tailwind/React/Vue output from a single image.

## What it is

A self-hostable web app that takes a screenshot of a UI (Figma export, browser screenshot, sketch) and generates equivalent HTML+Tailwind, React, or Vue code via vision-LLM models (GPT-4 Vision, Claude). Aimed at engineers and designers who want a faster path from visual mockup to working code. Hosted version at screenshottocode.com; OSS code here is the canonical reference.

## Key features

- Multi-output: HTML+Tailwind, React, Vue, Bootstrap.
- Iterative refinement — adjust the prompt and re-generate after seeing first output.
- Vision-LLM provider abstraction (OpenAI GPT-4V, Claude, Gemini).
- Self-hostable (Docker) or use the paid hosted SaaS.
- MIT-licensed.

## Tech stack

- TypeScript primary; React on the frontend; Python on the backend.
- Vision-LLM API integration as the core generator.

## When to reach for it

- You have UI mockups and want a faster starting point than hand-coding.
- You're prototyping internal tools and want LLM-driven scaffolding.

## When *not* to reach for it

- You need pixel-perfect, designer-approved output — the LLM output is approximate.
- You want vendor-free — relies on commercial vision-LLM APIs.

## Maturity signal

72k stars, 9k forks, MIT, ~2 years old. Active development. The 123 open-issues count is moderate.

## Alternatives

- `vercel/v0` — first-party Vercel "generate UI from prompt" with stronger integration.
- Figma → code plugins (Anima, Locofy) — use when starting from Figma rather than a screenshot.

## Tags

artificial-intelligence, large-language-model, typescript, react, vue, code-generation, ui, tailwindcss, vision-model
