# Zoveli Code

> **Status: planning.** This repository currently contains the integration plan
> only. The OpenCode source import and all code changes begin after the plan is
> approved.

Zoveli Code is the planned local-first coding agent for the
[Zoveli AI](https://github.com/blessandy9104-hue/zoveli-ai) project. It is a
Zoveli-branded derivative of [OpenCode](https://github.com/anomalyco/opencode),
MIT licensed, designed to connect the Zoveli math, memory, and trust surfaces to
a full coding-agent runtime.

## Why

Zoveli AI already ships a local-first assistant with verified math, local
memory, a vault, and an MCP tool surface. OpenCode already ships a mature
coding-agent runtime: a terminal UI, durable sessions, provider/model catalog,
permissions, plugins, LSPs, formatters, MCP servers, and worktrees.

Zoveli Code combines them:

- **OpenCode runtime** for coding, shell, file edits, providers, and sessions.
- **Zoveli MCP bridge** for verified math (`mathcore_*`, `math_*`) and local
  memory/vault tools.
- **Zoveli trust model** for verification receipts, audit trails, and
  local-first defaults.
- **Zoveli branding** for the product surface, installers, and documentation.

## Principles

- **The user owns the machine.** Every capability the project ships is
  available to its owner. We do not add paternalistic gates, hidden limits, or
  mandatory approvals. Permission policies exist so the owner can restrict
  themselves if they want to, not so the product can restrict the owner.
- **Local-first by default, never local-only by force.** Local models and local
  tools work without a network; cloud providers and remote MCP remain available
  when the owner configures them.
- **Verification is additive.** Verified math is shown explicitly, but the
  agent never pretends an unverified result is verified.
- **No telemetry, no training on user data, no lock-in.**

The CLI binary is `zoveli.ky`; the product name is **Zoveli Code**.

## Repository relationship

| Repository | Role |
| --- | --- |
| [`blessandy9104-hue/zoveli-ai`](https://github.com/blessandy9104-hue/zoveli-ai) | Product monorepo: runtime, mathcore, memory, vault, desktop, CLI. |
| [`anomalyco/opencode`](https://github.com/anomalyco/opencode) | Upstream open-source coding agent this repository derives from. |
| `blessandy9104-hue/zoveli-code` (this repository) | Zoveli-branded OpenCode derivative and Zoveli integration plugin. |

## Planned shape

```text
Zoveli Code CLI / TUI (binary: zoveli.ky, OpenCode fork)
  -> Zoveli plugin (@zoveli/code-plugin)
    -> local MCP bridge (scripts/zoveli-mcp-server.mjs)
      -> Zoveli local server (127.0.0.1:8080)
        -> mathcore / memory / vault / local tools
```

The integration is deliberately kept at the MCP and plugin boundary. Zoveli's
monorepo remains the source of truth for math and memory; Zoveli Code remains a
maintainable fork of OpenCode.

## Read the plan

- [PLAN.md](PLAN.md) — architecture, phases, milestones, risks, and open
  decisions.
- [NOTICE](NOTICE) — upstream attribution.

## License

MIT. This project includes software derived from OpenCode, Copyright (c) 2025
opencode. See [LICENSE](LICENSE) and [NOTICE](NOTICE).

Zoveli Code is not affiliated with or endorsed by the OpenCode maintainers.
