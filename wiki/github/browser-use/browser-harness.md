# browser-use/browser-harness

A thin, editable CDP harness that connects an LLM directly to a real Chrome browser and writes the helper code it is missing as it runs.

## What it is

Browser Harness gives an agent one WebSocket connection to a running Chrome/Chromium instance over the Chrome DevTools Protocol, with almost nothing between the model and the browser. It is aimed at people who want an agent to complete arbitrary browser tasks with full freedom rather than through a fixed automation API. When a helper it needs is missing, the agent writes that helper into an editable workspace file during execution, and reusable site-specific "domain skills" accumulate over time. Installation is done by pasting a setup prompt into a coding agent such as Claude Code or Codex.

## Key features

- Direct CDP attachment: plain `browser-harness` helper calls attach to a running Chrome via its remote-debugging endpoint; for isolated runs you launch Chrome with `--remote-debugging-port` and pass `BU_CDP_URL`.
- Self-editing workspace: the agent writes missing helper code into `agent_helpers.py` and creates domain skills under `domain-skills/` in the XDG config directory.
- Small protected core: the README describes roughly 1k lines across four core files, with a protected `src/browser_harness/` package kept separate from the agent-editable files.
- Domain skills, opt-in via `BH_DOMAIN_SKILLS=1`, teaching the agent selectors and flows for specific sites; the repo ships many examples (github, linkedin, amazon, and dozens more).
- Optional Browser Use Cloud browsers for stealth, sub-agents, or headless deployment, with a free tier.
- A `./browser-harness` launcher for running the working tree directly during local development.

## Tech stack

- Python, requiring >= 3.11 (classifiers list 3.11 and 3.12).
- Dependencies pinned exactly: cdp-use 1.4.5, fetch-use 0.4.0, pillow 12.2.0, websockets 15.0.1.
- Communicates with Chrome over the Chrome DevTools Protocol via WebSockets; no Playwright or Selenium layer.
- Packaged with setuptools (src layout), exposing a `browser-harness` console script; distributed on PyPI at version 0.1.7.
- Ships a Claude Code plugin manifest (`.claude-plugin/`) and a registerable skill.
- MIT licensed.

## When to reach for it

- Letting an agent drive your own logged-in browser session to complete open-ended tasks across arbitrary sites.
- Tasks where a fixed automation API gets in the way and you want the agent to write and improve its own helpers.
- Building up reusable, site-specific automation skills that persist and improve across runs.
- Scraping or acting on sites where reusing an existing authenticated Chrome profile matters.

## When *not* to reach for it

- It is Alpha (0.1.x, "Development Status :: 3 - Alpha") — not a stable API to build production automation against without expecting change.
- It requires driving a real Chrome over CDP with an LLM agent authoring helpers; for deterministic, hand-written scripts a conventional framework is simpler.
- The model rewriting helper code during execution is inherently less predictable than a fixed test script, a poor fit for tightly controlled CI automation.
- Attaching an agent to your real browser session grants broad access to logged-in accounts; unsuitable where that blast radius is unacceptable.

## Maturity signal

The repository is only a few months old (created April 2026) yet has drawn a large audience and is pushed daily, so it is early but very actively developed. The pyproject marks it Alpha at version 0.1.7 with pinned dependencies, and the architecture is deliberately tiny (~1k lines), signalling a young, fast-iterating tool rather than a settled framework. The permissive MIT license and an open contribution model (agent-generated domain skills submitted via PR) fit its experimental, community-driven posture. Expect rapid change and a still-maturing API.

## Alternatives

- browser-use — the same organization's main library; reach for it when you want a higher-level, more structured browser-agent library rather than a minimal editable CDP harness.
- Playwright — use it when you want a mature, deterministic browser-automation API with hand-written scripts and strong cross-browser support.
- Selenium — use it when you need broad language and browser coverage for conventional, scripted automation.

## Notes

- Skills are meant to be written by the harness, not by hand — the contributing guide asks people to run tasks and let the agent file the skill, then PR the generated folder, because agent-generated skills reflect what actually works in the browser.
- Installation is unusual: you paste a natural-language setup prompt into a coding agent, which installs the tool, registers a skill, and connects to your browser, including ticking Chrome's remote-debugging checkbox and clicking Allow on the per-attach popup on Chrome 144+.

## Tags

python, browser-automation, cdp, chrome, llm, ai-agent, web-scraping, cli
