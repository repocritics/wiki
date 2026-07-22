# datawhalechina/easy-vibe

A beginner-to-advanced "vibe coding" course that teaches building real apps by describing them to AI, delivered as an interactive multilingual website.

## What it is

Easy-Vibe is an open-source tutorial course from Datawhale about building software with AI, or "vibe coding." It guides learners from product prototyping through full-stack development to advanced Claude Code and agent workflows, using interactive visual components rather than plain text. It is aimed at complete beginners, product managers and founders, and developers who want to upgrade their AI-assisted workflow.

## Key features

- Three staged tracks: Stage 1 product prototyping, Stage 2 full-stack development (frontend, backend, billing, and capstone SaaS projects), and Stage 3 advanced AI-native engineering (Claude Code, MCP, Skills, Agent Teams, spec coding, cross-platform delivery).
- Interactive animated demos for concepts such as RAG, the terminal, AI IDE workflows, and image diffusion, built as Vue components.
- An appendix knowledge base covering a stated 9 major knowledge areas and 80+ interactive topics.
- Full multilingual coverage: all Stage 1–3 content is available in 10 languages.
- Cross-platform project tutorials including WeChat Mini Program, Android, iOS, PWA, a browser extension, Electron, Qt, and a VS Code extension.

## Tech stack

- Documentation site built with VitePress (^2.0.0-alpha.16) and Vue (^3.5).
- Node.js ≥ 18; content authored in Markdown with embedded Vue interactive components.
- UI and visualization libraries: Element Plus, Mermaid, reveal.js, TypeIt, and Viewer.js.
- Tooling: ESLint 9, Prettier, and Husky; deployment configured via a GitHub Pages workflow (with Vercel referenced in the README).
- Content license CC-BY-NC-SA-4.0.

## When to reach for it

- You are new to programming and want to build a first app by describing it to AI.
- You want a structured path that moves from prototype to full-stack to agent workflows.
- You want visual, interactive explanations of concepts such as RAG, authentication, and APIs.
- You want the material in a language other than English — 10 are supported.

## When *not* to reach for it

- You want a code library or framework — this is course content, not software to import.
- You want vendor-neutral computer-science theory — the material is practice-first and tool-oriented.
- You need a permissive license for reuse — CC-BY-NC-SA-4.0 forbids commercial use and requires share-alike.
- You are an experienced engineer looking for reference documentation rather than a guided beginner course.

## Maturity signal

Created in late December 2025, Easy-Vibe reached roughly 18,400 stars within months, was last pushed in July 2026, and has only 16 open issues, indicating an actively maintained and rapidly popular course with frequent content updates, including the completion of full multilingual coverage in mid-2026. Because it is a content project under CC-BY-NC-SA-4.0, its maturity is best read as the breadth and freshness of the curriculum rather than software API stability.

## Alternatives

- freeCodeCamp — use it when you want a broad, traditional coding curriculum rather than an AI-first approach.
- The Odin Project — use it when you want a structured full-stack path without the AI-agent focus.
- Official Claude Code or Cursor documentation — use them when you want reference material for a specific tool rather than a guided course.

## Notes

- Although the repository's primary language is reported as JavaScript, it is a VitePress documentation and course site rather than an application, and most files are Vue interactive-demo components.
- The `package.json` lists a `claude` npm dependency alongside the site tooling.
- The content license (CC-BY-NC-SA-4.0) is unusual for a repository tagged as a code project.
- Datawhale links sibling courses from the README, including Hands-on Modern RL and Learn Harness Engineering.

## Tags

javascript, vue, vitepress, tutorial, course, vibe-coding, llm, education, documentation, mcp, multilingual
