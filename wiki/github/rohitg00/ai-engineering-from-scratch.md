# rohitg00/ai-engineering-from-scratch

A free, open-source AI engineering curriculum that builds every algorithm from raw math first, then reruns it through production libraries, across 503 lessons.

## What it is

AI Engineering from Scratch is a structured curriculum that takes you from linear algebra to autonomous agent swarms by implementing each algorithm by hand before touching a framework. It spans 20 phases and 503 lessons in Python, TypeScript, Rust, and Julia, with every lesson producing a reusable artifact — a prompt, a skill, an agent, or an MCP server. It is for people who want to understand how AI actually works end to end, not just call APIs, and can run everything on their own laptop.

## Key features

- 503 lessons organized into 20 stacked phases, from math foundations up through deep learning, vision, NLP, transformers, LLMs, agents, and production.
- A "Build It / Use It" structure per lesson: implement the algorithm from raw math with no frameworks, then run the same thing through a production library such as PyTorch or scikit-learn.
- A consistent lesson layout (`code/`, `docs/en.md`, `outputs/`) so each lesson ships a runnable implementation plus a reusable prompt, skill, agent, or MCP server.
- Built-in agent skills — `/find-your-level` for a placement quiz and personalized path, and `/check-understanding <phase>` for per-phase quizzes — usable inside Claude, Cursor, Codex, and other agents that read `SKILL.md`.
- A companion website and a script (`scripts/install_skills.py`) to install the produced artifacts into your own workflow.

## Tech stack

- Primary language: Python; lessons also include TypeScript, Rust, and Julia code samples.
- Core Python dependencies (from `requirements.txt`) include numpy `>=1.24`, torch `>=2.0`, torchvision/torchaudio, transformers `>=4.30`, datasets, tokenizers, accelerate, scikit-learn `>=1.3`, pandas `>=2.0`, librosa, tiktoken, plus the anthropic `>=0.25` and openai `>=1.0` clients.
- Jupyter for notebooks and Docker are covered as part of the tooling phase.
- A static site is generated from the repository (stats and figures are build-generated). License: MIT.

## When to reach for it

- Learning AI engineering by building each component yourself rather than only consuming tutorials or calling APIs.
- Filling gaps between scattered papers, posts, and demos with a single linear path from math to production.
- Producing a portfolio of reusable prompts, skills, agents, and MCP servers while you learn.
- Placing yourself into the right starting phase and self-quizzing with the bundled agent skills.

## When *not* to reach for it

- You need a production-ready library or framework to depend on; this is educational material, not a package to import.
- You want fast, top-down practical results and are not interested in deriving algorithms from first principles.
- You lack the time budget the curriculum assumes (roughly 320 hours) or do not want a from-scratch, code-it-yourself approach.

## Maturity signal

The repository is very new, created in early 2026, yet already carries a very large star count, which signals strong interest in the format. The last push is recent relative to that creation date, and the MIT license is permissive with the repository not archived. Because it is an early-stage, large-scope curriculum — the README points readers to "any completed lesson" — expect that not every one of the 503 lessons is finished, so treat it as an actively growing work in progress rather than a settled reference.

## Alternatives

- fast.ai — reach for it when you want a top-down, practical deep learning course that gets you to working models quickly.
- Andrej Karpathy's "neural networks: zero to hero" — reach for it when you want a focused, from-scratch build-up of neural nets and language models rather than a full 20-phase span.
- Hugging Face courses — reach for these when you want library-first, transformers-and-NLP training rather than raw-math implementations.
- Dive into Deep Learning (d2l.ai) — reach for it when you want an interactive textbook with runnable notebooks across frameworks.

## Notes

- Every lesson deliberately ends with a shippable tool rather than a "congratulations, you learned X," so the curriculum doubles as a way to accumulate 503 usable artifacts.
- It comes from the creator of the `agentmemory` project, and the README opens with the framing that 84% of students use AI tools but only 18% feel prepared to use them professionally.

## Tags

python, typescript, rust, julia, machine-learning, deep-learning, llm, ai-engineering, course, tutorial, from-scratch, transformers
