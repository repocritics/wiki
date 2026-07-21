# Leonxlnx/taste-skill

A collection of portable Agent Skills that steer AI coding agents toward better-looking frontends instead of generic, boilerplate UI.

## What it is

Taste Skill is a set of `SKILL.md` instruction files that you load into AI coding agents such as Codex, Cursor, or Claude Code to improve the design quality of the interfaces they generate. Each skill encodes rules for layout, typography, motion, and spacing so that agent-built UIs look less templated. The repository also ships image-generation skills that produce reference boards (web comps, mobile flows, brand kits) meant to be paired with a generator like ChatGPT Images before handing the frames to a coding agent. It is aimed at developers who build frontends with AI agents and want more control over the visual result.

## Key features

- Multiple specialized skills, each doing one job: a general default (`design-taste-frontend`), a stricter GPT/Codex variant (`gpt-taste`), a redesign auditor for existing projects, and fixed-direction styles (soft, minimalist, brutalist).
- Adjustable 1–10 dials in the main skill for DESIGN_VARIANCE (layout experimentation), MOTION_INTENSITY (animation depth), and VISUAL_DENSITY (information per viewport).
- Framework-agnostic rules that target design intent rather than a single API, so they apply to React, Vue, or Svelte.
- Separate image-generation skills (`imagegen-frontend-web`, `imagegen-frontend-mobile`, `brandkit`) that output reference images only, plus an `image-to-code` skill that runs generate → analyze → implement as one workflow.
- Installable through the `npx skills add` CLI, or by copying a `SKILL.md` into a project or pasting it into a chat.
- The default skill (v2, experimental) includes brief inference, a design-system map, a hard em-dash ban, canonical GSAP code skeletons, a redesign-audit protocol, and a pre-flight check.

## Tech stack

- Primary language reported as JavaScript; the repository's code is limited to Node.js `.mjs` scripts under `scripts/` used to process README and sponsor assets.
- Core deliverables are Markdown `SKILL.md` files under `skills/`, following the Agent Skills format compatible with `vercel-labs/agent-skills`.
- Distributed via the `npx skills add` CLI, which scans the `skills/` folder.
- Ships a Claude Code plugin manifest (`.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`).
- Motion guidance references GSAP; the `stitch-skill` supports a Google Stitch-compatible `DESIGN.md` export.
- No recognized root build manifest (no `package.json` at the repo root); MIT licensed.

## When to reach for it

- You generate frontends with Codex, Cursor, or Claude Code and the output looks generic or templated.
- You want to nudge design decisions (layout variance, motion, density) without hand-writing every rule.
- You need to redesign or audit an existing UI rather than start from scratch (`redesign-existing-projects`).
- You have a chosen visual direction and want a matching preset (soft/high-end, minimalist/editorial, brutalist/industrial).
- You want to generate reference comps or brand boards first and then implement them in code.
- The agent keeps shipping half-finished work and you want full-output enforcement (`full-output-enforcement`).

## When *not* to reach for it

- You are not working through an AI agent that can load skill instructions; these are prompt/instruction files, not a runtime library or component kit.
- Your work is backend or non-UI; the skills target frontend layout, typography, motion, and spacing.
- You need vetted, reusable component code with guaranteed behavior; results here depend on the underlying model and vary between runs.
- You require deterministic, pixel-exact output, since the skills steer a language model rather than render fixed designs.

## Maturity signal

Despite being created only a few months before its last recorded push, the project has accumulated a very large following (tens of thousands of stars), which points to fast, viral adoption rather than a long track record. It is actively maintained: the default skill was recently rewritten as an experimental v2 that is still iterating toward a stable release, with the previous version preserved under a separate install name for anyone who depends on it. The MIT license and the not-archived status make it low-risk to adopt, but the "experimental" labeling on the flagship skill means behavior can still change between updates.

## Alternatives

- Use **shadcn/ui** when you want curated, copy-in component code with predictable output instead of instructions that steer an agent's design taste.
- Use **v0 by Vercel** when you prefer a hosted AI tool that generates UI directly rather than skills layered onto your own agent.
- Use **Google Stitch** when you want a dedicated AI UI-design tool; Taste Skill's `stitch-skill` is built to be compatible with it rather than replace it.
- Use the **vercel-labs/agent-skills** ecosystem directly when you want to author or mix your own skills rather than adopt this opinionated anti-slop set.

## Notes

- Install names differ from folder names; you pass the `name:` value from the SKILL frontmatter (for example `design-taste-frontend`, not `taste-skill`) to `--skill`.
- The image-generation skills deliberately produce images only and are meant to be paired with an external generator, then fed back to a coding agent.
- The README includes a disclaimer that there is no official token or crypto associated with the project, a sign its visibility has attracted unaffiliated scam tokens.
- Background research that informed the anti-repetition rules lives in the `research/` directory, including material on model "laziness" and output limits.

## Tags

javascript, agent-skills, llm, prompt-engineering, frontend, ui-design, claude-code, codex, design, motion, framework-agnostic
