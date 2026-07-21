# HKUDS/CLI-Anything

A generator that wraps existing software in structured, agent-friendly command-line interfaces so AI coding agents can drive it.

## What it is

CLI-Anything is a Python toolchain for turning arbitrary software — desktop apps, backend services, or source repositories — into command-line harnesses that AI agents can call. It targets developers who want their AI coding agent (Claude Code, Codex, OpenClaw, and others) to operate tools that only expose a GUI or an API by giving those tools a consistent, self-describing CLI. Alongside the generator it ships CLI-Hub, a package manager for browsing, installing, and managing community-built CLIs.

## Key features

- A seven-phase generator pipeline (analyze, design, implement, plan tests, write tests, document, publish) that produces a Click-based Python CLI with a REPL, JSON output, and undo/redo.
- CLI-Hub package manager (`pip install cli-anything-hub`) with `list`, `search`, `info`, `install`, `update`, `uninstall`, and `launch` commands for both first-party and public third-party CLIs.
- Every generated CLI ships a `SKILL.md` definition so skills-compatible agents can discover and install it.
- Plugin or skill integrations for Claude Code, Pi, OpenCode, OpenClaw, Codex, Hermes, Reasonix, Qodercli, GitHub Copilot CLI, and Goose.
- A `refine` command that performs gap analysis against a target's full capabilities and incrementally adds commands, tests, and docs.

## Tech stack

- Python 3.10+ (primary language).
- Click 8.0+ for the generated CLIs.
- pytest for the unit and end-to-end test suites; generated harnesses publish via `setup.py`.
- Distributed as a Claude Code plugin marketplace (`.claude-plugin/marketplace.json`) and as per-agent skill and extension installers.

## When to reach for it

- You want an AI agent to operate desktop software (GIMP, Blender, LibreOffice, QGIS, FreeCAD, and similar) that has no agent-native interface.
- You need a repeatable way to generate and test CLIs for internal tools or codebases.
- You already work inside a supported agent platform and want a one-command way to add tool coverage.
- You want to install existing community CLIs rather than build your own.

## When *not* to reach for it

- Your target software already exposes a stable API or MCP server that agents can call directly.
- You are not working through an AI coding agent — the value is agent-driven CLI generation, not hand-authored tools.
- You need a language other than Python for the generated harness; output is Click and Python.
- You want a single small dependency; this is a large, actively growing monorepo of many harnesses.

## Maturity signal

Created in early March 2026 and pushed within the last few weeks, the project is very young but moving quickly, with a large star count and a steady stream of merged CLI harnesses and fixes documented in the README's news log. Apache-2.0 licensing and the volume of contributor-submitted harnesses point to an actively maintained, community-driven effort rather than a settled, stable release. Expect rapid change and evolving conventions.

## Alternatives

- Model Context Protocol servers — use MCP directly when your tool can expose a standardized protocol endpoint instead of a generated CLI.
- Open Interpreter — use it when you want an agent to write and run code against a tool ad hoc rather than through a pre-built, tested harness.
- Hand-written Click or Typer CLIs — use these when you need full control over a single tool and do not want a generator's conventions.

## Notes

The README documents a large and growing catalog of harnesses (GIMP, Blender, QGIS, FreeCAD with 258 commands, Calibre, Zotero, and many more) plus repeated security hardening passes (path-traversal fixes, `defusedxml` for untrusted XML), suggesting the generated CLIs are treated as real software with test and security expectations rather than throwaway wrappers.

## Tags

python, cli, ai-agents, code-generation, developer-tools, llm, automation, skills
