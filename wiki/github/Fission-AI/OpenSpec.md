# Fission-AI/OpenSpec

A Spec-Driven Development toolkit for AI coding assistants — structured specifications + planning artifacts that guide agent execution.

## What it is

A TypeScript-based methodology + tooling package for Spec-Driven Development (SDD) targeted at AI coding workflows. Provides spec templates, planning scaffolds, and integration points for Claude Code, Cursor, and similar agent surfaces. Operates in the same space as `github/spec-kit` — both pitch structured specs as the bridge between product intent and AI-generated code. MIT-licensed.

## Key features

- Spec templates for product intent, design decisions, and implementation plans.
- Context-engineering guidance for AI agents — what to give them, in what shape.
- TypeScript CLI for spec management.
- Integration with major coding-agent surfaces.
- Companion site at openspec.dev.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- CLI distributed via npm.

## When to reach for it

- You want a third-party SDD methodology rather than GitHub's spec-kit.
- You're comparing the SDD-tooling space and want hands-on exposure to multiple options.
- You're already deep into Cursor / Claude Code and want a framework-light spec layer.

## When *not* to reach for it

- You want first-party GitHub tooling integration — `github/spec-kit` is closer-fit.
- You don't want a methodology framework — plain Markdown specs in your repo work.

## Maturity signal

52k stars, 4k forks, MIT, actively maintained. The SDD-tooling category is young; multiple frameworks competing for the same niche.

## Alternatives

- `github/spec-kit` — first-party from GitHub.
- Linear / Notion + manual prompts — use when you don't want a methodology framework.

## Tags

artificial-intelligence, large-language-model, agent, typescript, spec, methodology, mit-license, developer-tools
