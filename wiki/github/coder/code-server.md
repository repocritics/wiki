# coder/code-server

VS Code in the browser — host VS Code on a server, access it from any browser. The OSS substrate for Coder (the company) and a popular self-hosted developer-environment tool.

## What it is

A package that runs the VS Code editor as a web server, with a browser frontend that talks to the same Monaco-based UI you'd get locally. Designed for cloud dev environments, remote pair programming, and "code from anywhere" use cases. Coder Inc. (the company) layers commercial features on top; the OSS core is fully usable standalone.

## Key features

- Full VS Code UX in a browser — same editor, terminal, extensions.
- Extension support via Open VSX Registry (Microsoft Marketplace is licensed to Microsoft).
- Multi-user workspaces (separate code-server processes per user).
- Self-host via Docker / direct install / Coder platform.
- Works with reverse proxies, auth headers, TLS termination.
- MIT-licensed.

## Tech stack

- TypeScript primary.
- Builds on the OSS VS Code core (`microsoft/vscode`).
- Node.js runtime on the server side.

## When to reach for it

- You want a cloud dev environment without managing IDEs locally.
- You're enabling Chromebooks / iPads / tablets for development work.
- You're standing up internal dev environments for fleets and want a familiar editor.

## When *not* to reach for it

- You want first-party Microsoft VS Code Server (GitHub Codespaces) — use Codespaces if you're already on GitHub.
- You need access to the Microsoft Marketplace's extensions — code-server uses Open VSX, which has overlapping but distinct catalog.

## Maturity signal

78k stars, 6.6k forks, MIT, actively maintained under Coder Inc. Open-issues count of 153 is low for the surface area.

## Alternatives

- GitHub Codespaces — first-party.
- Gitpod — alternative commercial cloud dev environment.
- Project IDX — Google's alternative.
- JetBrains Gateway — for JetBrains-flavored remote dev.

## Tags

typescript, vscode, ide, remote-development, cloud-development, self-hosted, mit-license
