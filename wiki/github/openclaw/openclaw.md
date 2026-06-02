# openclaw/openclaw

A self-hosted personal AI assistant that lives across messaging channels and renders an interactive Canvas on macOS, iOS, and Android.

## What it is

A single-user assistant runtime built around a "Gateway + channels + skills" architecture. The Gateway is the control plane; the product surface is the assistant itself, which speaks and listens via your existing apps (WhatsApp, Telegram, Slack, Discord, iMessage, Teams, Matrix, Signal, LINE, WeChat, and ~15 more) instead of a dedicated client. A live Canvas extends the conversation with visual state the user controls. Intended for users who want a personal, always-on assistant that runs on their own machines.

## Key features

- Channel adapters for 24+ messaging surfaces (WhatsApp, Telegram, Slack, Discord, Google Chat, Signal, iMessage, IRC, Microsoft Teams, Matrix, Feishu, LINE, Mattermost, Nextcloud Talk, Nostr, Synology Chat, Tlon, Twitch, Zalo, WeChat, QQ, WebChat, and others).
- Native voice on macOS / iOS / Android — speak and listen, not just type.
- Live Canvas: a renderable, user-controlled visual layer adjacent to chat.
- CLI-driven onboarding (`openclaw onboard`) walks through Gateway + workspace + channels + skills setup.
- Multi-runtime support: npm, pnpm, and bun. Docker and Nix install paths documented.
- Skills are pluggable — the assistant extends via the same skill manifest used across the platform.

## Tech stack

- Primary language: TypeScript.
- Distribution: npm package + Docker image + Nix package (`openclaw/nix-openclaw`).
- Local-first orientation: Gateway runs on the user's own machine; WSL2 is the recommended Windows host.
- Sponsorship-funded development (OpenAI, GitHub, NVIDIA, Vercel, Blacksmith listed in README).

## When to reach for it

- You want a personal AI assistant that follows you across the messaging apps you already use, instead of opening another standalone app.
- You want voice in/out on mobile without surrendering the conversation transcript to a third-party assistant service.
- You're comfortable running a self-hosted control plane and configuring channel credentials per integration.

## When *not* to reach for it

- You want a fully managed, install-and-forget cloud assistant — the Gateway is self-hosted and the channel credentials are yours to manage.
- You need a multi-tenant team assistant — OpenClaw is explicitly single-user.
- You're allergic to fast-moving codebases: 7,000+ open issues and a six-month-old project signal active churn.

## Maturity signal

376k stars in roughly six months puts this in the GitHub-trending fast-lane. 78k forks indicate real downstream usage rather than drive-by stargazing. The 7k open-issues count is high in absolute terms but proportional to the velocity — this is a project being shaped in public, not in long-tail maintenance mode. License declared MIT in README but reported as `NOASSERTION` on the GitHub side; downstream packagers should verify the LICENSE file directly before redistributing.

## Alternatives

- `hass-io` / Home Assistant — use when you want IoT-and-home-automation focus rather than a chat-first assistant.
- LangChain / LlamaIndex hosted apps — use when you're building an agent yourself rather than running a turnkey assistant.
- Apple Shortcuts / native Siri — use when you only need device-local automations without cross-channel reach.

## Notes

The "Gateway is the control plane" framing matters for security review: channel adapters are credentialed gateways, so a compromise of the Gateway machine exposes everything plugged into it. The README emphasizes a strong on-device posture (no central hosted service) but doesn't yet ship an opinionated hardening guide. The 24-channel surface is the project's most distinctive bet; the trade-off is that each adapter is one more attack surface and one more compatibility burden.

## Tags

artificial-intelligence, personal-assistant, self-hosted, typescript, command-line-interface, voice, chatbot, telegram, slack, discord, ios, android, macos, open-source
