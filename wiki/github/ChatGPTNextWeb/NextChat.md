# ChatGPTNextWeb/NextChat

A cross-platform LLM chat client — supports OpenAI, Claude, Gemini, Ollama, Groq, and more, with desktop builds via Tauri and Vercel-hostable web deployments.

## What it is

A TypeScript + Next.js chat application that fronts the major LLM providers (OpenAI, Claude, Gemini, Groq, local-via-Ollama) plus a Tauri-packaged desktop build. One-click Vercel deploy for the web variant. Aimed at users who want one polished chat client across providers rather than installing one app per LLM.

## Key features

- Multi-provider: OpenAI, Anthropic Claude, Google Gemini, Groq, Ollama (local), Azure OpenAI.
- Web + desktop (Tauri) + iOS / Android / Linux / Windows / macOS distributions.
- Prompt-mask system (saved system prompts as personas).
- Plugin / tool integration.
- One-click Vercel deploy as the easy hosting path.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- Next.js for the web app.
- Tauri for desktop builds.

## When to reach for it

- You want one chat client across multiple LLM providers.
- You're deploying internal chat tooling and want a clone-and-config-on-Vercel solution.

## When *not* to reach for it

- You want vendor-supported chat — go to the vendor's official apps (ChatGPT, Claude.ai, Gemini).
- You want enterprise SSO / audit — those are commercial features in vendor-managed tools.

## Maturity signal

88k stars, 60k forks, MIT, actively maintained. 60k forks reflect the "one-click deploy on Vercel" pattern — every user who deploys forks.

## Alternatives

- Vendor official apps.
- `open-webui/open-webui` — heavier feature set, self-hosted-first.
- LibreChat — strict-OSI alternative.

## Tags

typescript, large-language-model, chatbot, claude, openai, gemini, ollama, cross-platform, tauri, nextjs, vercel
