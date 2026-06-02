# code-yeongyu/oh-my-openagent

omo — an agent harness for complex codebases, TypeScript + TUI, positioning itself as "the pickaxe for complex software engineering".

## What it is

A TypeScript-implemented agent harness for coding tasks, distributed via a TUI (terminal UI) plus orchestration primitives across Claude, OpenAI Codex, Gemini, Cursor, OpenCode. Aimed at engineers working on larger / messier codebases where standalone coding agents struggle with context management. License is `NOASSERTION` — verify before commercial reuse.

## Key features

- Cross-provider: Claude (incl. claude-skills), OpenAI Codex, Gemini, Cursor, OpenCode.
- TUI front-end as the primary developer surface.
- Orchestration layer for multi-step / multi-agent workflows.
- Topic-list emphasis on agent-skills + IDE integration.

## Tech stack

- TypeScript primary.
- TUI rendering (likely via Ink or similar React-for-CLI libraries).

## When to reach for it

- You want a TUI-driven agent harness rather than a CLI-or-IDE-extension model.
- You're evaluating third-party multi-provider agent harnesses.

## When *not* to reach for it

- You want vendor-supported tooling — first-party agents are safer.
- You're allergic to non-OSI licenses — `NOASSERTION` is a yellow flag for commercial reuse.
- You're risk-averse about installing third-party agent harnesses with filesystem + terminal scope; audit before integration.

## Maturity signal

61k stars, 5k forks, last push the day this page was generated. The agent-harness space is fast-moving and prone to high-velocity stars without proportional usage signals. License absence warrants direct LICENSE inspection.

## Alternatives

- `obra/superpowers`, `affaan-m/ECC`, `mattpocock/skills` — comparable third-party agent / skills frameworks.
- `anthropics/claude-code` — first-party reference.
- `cline/cline` — VS-Code-flavored alternative.

## Tags

artificial-intelligence, large-language-model, agent, typescript, tui, claude, openai, gemini, framework, claude-skills
