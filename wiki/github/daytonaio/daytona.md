# daytonaio/daytona

Secure, elastic infrastructure for running AI-generated code — sandboxes for agent-produced code execution.

## What it is

A platform for running untrusted / AI-generated code in isolated sandboxes. Aimed at AI-agent products that need to safely execute model output without exposing the host: spin up an ephemeral container, run the code, capture results, tear down. Pivoted from a developer-environment focus to AI sandboxing as the market matured.

## Key features

- Per-request sandbox provisioning with strong isolation.
- Multi-language runtime support — Python, JavaScript, shell, more.
- Auto-cleanup of completed sandboxes.
- SDK + REST API for integration.
- Self-hostable or use Daytona Cloud.
- AGPL-3.0 licensed.

## Tech stack

- TypeScript primary on the orchestration layer.
- Container-based sandboxing (likely Docker / Firecracker / similar).

## When to reach for it

- You're building an AI agent that generates and runs code, and need a sandbox layer.
- You're shipping a code-interpreter feature in a chat product.
- You're operating an LLM-based developer tool that needs safe execution.

## When *not* to reach for it

- You need vendor-supported with SLAs at enterprise scale.
- You can run trusted code in your own infra without isolation.
- AGPL-3.0 is incompatible with your commercial license model — verify.

## Maturity signal

73k stars, 5.6k forks, AGPL-3.0, actively maintained. Pivoted positioning from dev-env to AI-sandbox. Open-issues count of 430.

## Alternatives

- E2B (e2b.dev) — comparable AI-sandbox commercial / OSS product.
- Modal — code-execution-as-a-service.
- Custom Firecracker / gVisor + your own orchestration.

## Tags

artificial-intelligence, agent, code-execution, sandbox, typescript, agpl, infrastructure, developer-tools
