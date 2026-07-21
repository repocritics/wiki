# JuliusBrussee/caveman

A skill for Claude Code and 30+ other agents that makes the model answer in terse "caveman-speak," cutting output tokens while keeping code and commands exact.

## What it is

Caveman is a skill/plugin that tells an AI coding agent to drop filler and reply in tight fragments — "caveman-speak" — while leaving code, commands, and error text byte-for-byte unchanged. The README reports about 65% fewer output tokens on average across its benchmark prompts, on the premise that it shrinks what the agent *says*, not what it knows. It is aimed at developers who find agent replies needlessly verbose and want shorter, faster-to-read answers on every turn.

## Key features

- Compresses reply style, not substance, and explicitly preserves code, commands, and errors verbatim; it keeps the user's language rather than translating.
- Six intensity levels switchable with `/caveman <level>`: `lite`, `full` (default), `ultra`, and a `wenyan` mode that renders in classical Chinese for maximum meaning per token.
- Extra commands: `/caveman-commit` (Conventional Commit messages), `/caveman-review` (one-line PR comments), `/caveman-stats` (session token usage and lifetime savings), and `/caveman-compress` (rewrite a memory file like `CLAUDE.md` into caveman-speak to cut input tokens on future sessions).
- A `caveman-shrink` MCP middleware that wraps any MCP server and compresses its tool descriptions, plus `cavecrew-*` subagents (investigator, builder, reviewer).
- A single installer that detects the AI coding agents on your machine and installs for each; on Claude Code, Codex, and Gemini a hook turns it on from the first message.
- No telemetry, analytics, accounts, or backend; after install the skill makes zero network calls.

## Tech stack

- JavaScript; the installer package is `caveman-installer` (version 0.1.0), MIT licensed, requiring Node >=18.
- Installation via a `curl | bash` / PowerShell script, a Node `bin` (`caveman`), or per-agent paths (Claude Code plugin, Gemini CLI extension, `npx skills add`).
- Ships skills, commands, agents, and plugin bundles; local hook scripts (shell + PowerShell) drive session activation and the statusline; benchmarks/evals use Python (`benchmarks/run.py`, `evals/`).
- The `caveman-shrink` MCP middleware is a separate Node package under `src/mcp-servers/`.

## When to reach for it

- Agent replies are too long and you want shorter, denser answers without losing technical content.
- You want compressed commit messages and one-line PR review comments.
- You want to shrink a recurring memory/context file (like `CLAUDE.md`) so every future session loads a smaller prompt.
- You use several agents and want one installer to apply the same style everywhere.

## When *not* to reach for it

- You care about total per-session cost: the README is explicit that only output tokens shrink, the skill adds roughly 1–1.5k input tokens per turn, and on already-terse workloads savings can go net-negative.
- You want to reduce the actual amount of code the agent writes rather than the words it uses — that is a different tool (see Alternatives).
- You need verbose, explanatory output (teaching, detailed rationale), which this deliberately removes.
- Your host is one of the few agents the installer does not support.

## Maturity signal

The project is recent (created April 2026) and popular, with a very high star count, pushes as of early July 2026, and an MIT license; the installer package sits at an early 0.1.0. Committed benchmarks and evals, a documented "honest numbers" caveat, and a maintainer guide (`CLAUDE.md`) point to an actively maintained project that is candid about the limits of its headline metric rather than one coasting on a meme premise.

## Alternatives

- **[ponytail](https://github.com/DietrichGebert/ponytail)** — use it when the problem is the agent *over-building* rather than over-talking; it trims the code written, and the two are described as complementary.
- **[caveman-code](https://github.com/JuliusBrussee/caveman-code)** — use it when you want a whole terse coding agent end to end, not just a skill that compresses replies inside your existing agent.
- **The default verbose agent** — use it when you specifically need long, explanatory answers and are not optimizing for output length.

## Notes

- Savings apply to output tokens only; input and reasoning tokens are untouched, so whole-session numbers are smaller than the 65% headline and the docs point to a dedicated "honest numbers" writeup.
- The README cites a March 2026 paper (arXiv 2604.00025) reporting that constraining large models to brief answers improved accuracy on some benchmarks, as evidence that brevity can help correctness, not just cost.
- The `wenyan` level is an intentional exception to the "keep your language" rule because classical Chinese packs the most meaning per token.

## Tags

javascript, claude-code, prompt-engineering, ai-agents, llm, developer-tools, cli, agent-skills, mcp
