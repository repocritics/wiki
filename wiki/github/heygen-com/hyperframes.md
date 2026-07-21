# heygen-com/hyperframes

An open-source framework for turning HTML, CSS, media, and seekable animations into deterministic MP4 videos, designed so both humans and AI coding agents can author them.

## What it is

HyperFrames lets you define a video as an HTML file, annotate it with `data-*` attributes for timing and tracks, and render it to a reproducible MP4. The renderer seeks each frame in headless Chrome and encodes the output with FFmpeg, so the same input produces the same video every time. It targets developers building automated content pipelines and AI coding agents (Claude Code, Cursor, Gemini CLI, Codex) that already know how to write HTML but lack the domain patterns for video production. It runs three ways: locally through the CLI, from an agent via installable skills, or as the rendering core behind hosted authoring workflows such as HeyGen.

## Key features

- HTML-native compositions: a video is an `index.html` with `data-*` timing/track attributes and `class="clip"` elements, with no React requirement and no proprietary timeline format.
- Adapter-based animation: bring GSAP, CSS animations, Lottie, Three.js, Anime.js, WAAPI, or a custom frame adapter for seekable, frame-accurate motion.
- Deterministic rendering: frames are seeked in headless Chrome and encoded with FFmpeg so runs are reproducible, which the project positions for CI and regression testing.
- Agent skills: ships 19 skills (a `/hyperframes` router plus creation workflows like `/product-launch-video`, `/pr-to-video`, `/slideshow`, and domain skills) that teach coding agents the production loop of plan, write HTML, wire animations, add media, lint, preview, render.
- CLI dev loop: `init`, `preview` (live reload in the browser), `lint`, `check`, `render`, plus `add` for installing catalog blocks and components.
- Distributed rendering: local rendering plus an AWS Lambda render path and HeyGen-hosted cloud rendering.
- Monorepo of composable packages: CLI, core, engine (Puppeteer + FFmpeg capture), producer (capture/encode/audio mix), studio editor, embeddable player web component, shader transitions, and an AWS Lambda SDK.

## Tech stack

- TypeScript (`^5.0.0`); primary repository language is TypeScript.
- Node.js `>=22` required at runtime.
- Bun as the workspace build/test runner (`bun.lock`, npm workspaces under `packages/*`).
- FFmpeg required for encoding; Puppeteer / headless Chrome for frame capture.
- Animation runtimes via adapters: GSAP, Lottie, Three.js, Anime.js, CSS, WAAPI.
- React `^19.0.0` pinned via `resolutions`/`overrides` (used by the studio editor UI).
- Tooling: oxlint (`^1.56.0`), oxfmt (`^0.41.0`), tsx (`^4.21.0`), knip, lefthook, commitlint, happy-dom for tests.
- License: Apache-2.0.

## When to reach for it

- You want an AI coding agent to generate videos and prefer it to write plain HTML rather than a React project.
- You need reproducible, deterministic renders for CI, regression tests, or automated content pipelines.
- You are building product launch videos, PR walkthroughs with animated diffs, data visualizations and chart races, captioned social clips, or docs/PDF-to-video explainers.
- You want to preview a composition directly in the browser with no build step and render the same file locally or on AWS Lambda.
- You need seekable, frame-accurate animation and want to keep using a familiar runtime like GSAP or CSS rather than learning a bespoke timeline format.

## When *not* to reach for it

- Your team's authoring model is React components and you would rather compose with JSX; the README frames that as Remotion's bet, not HyperFrames'.
- You need a mature, batteries-included managed cloud renderer; the README describes local and AWS Lambda paths, and notes Remotion Lambda as the more established cloud renderer.
- You want a conventional non-linear video editor for hand-editing footage rather than authoring compositions in code.
- You cannot run Node.js 22+, FFmpeg, and a headless Chrome capture step in your environment.
- You need simple stream-level operations like trimming or concatenating existing clips, where calling FFmpeg directly is lighter than an HTML composition engine.

## Maturity signal

HyperFrames is young but moving quickly: created in March 2026, it had reached roughly 36.5k stars and 3.4k forks by mid-2026, with the last push a day before this snapshot. That combination points to active maintenance and rapid adoption rather than a settled, long-tail project. It is not archived, uses the permissive Apache-2.0 license, and is stated to run in production at HeyGen with named community adopters. The open issue count (around 245) is consistent with a fast-growing codebase under heavy use; the monorepo also marks several pieces (for example the studio) as "available, evolving," so expect surface area to keep shifting.

## Alternatives

- Remotion: reach for it instead when your team wants to author compositions as React components and values a mature cloud renderer, accepting its source-available license.
- Motion Canvas: reach for it instead when you want a TypeScript, code-driven animation tool with its own scene API rather than HTML-native compositions.
- FFmpeg (directly): reach for it instead when your task is trimming, concatenating, or transcoding existing media and you do not need HTML-authored, animated compositions.

## Notes

- The skill install flow has sharp edges: interactive `skills add` opens a picker with the "Core Skills" group and nothing pre-selected, but a non-interactive or agent run without `--skill` installs all 19; the README recommends `npx hyperframes skills update` for agents and stresses keeping `--full-depth`, because without it `skills add` pulls a skills.sh registry blob that lags `main` by hours.
- The repo uses Git LFS for about 240 MB of golden regression-test baseline `.mp4` files under `packages/producer/tests`; contributors should install Git LFS first, or clone with `GIT_LFS_SKIP_SMUDGE=1` for source only.
- `frame.md` is a distinct concept: a translation layer that inverts a web-oriented `design.md` into a `DESIGN.md` superset an agent can compose video from without guessing at scale.
- The project ships plugin manifests for multiple agent ecosystems (`.claude-plugin`, `.codex-plugin`, `.cursor-plugin`) and mirrors skills under both `.agents/skills` and `.claude/skills`.

## Tags

typescript, video, rendering, framework, html, animation, ffmpeg, puppeteer, ai-agents, mcp, cli, monorepo
