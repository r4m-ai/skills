# r4m.ai Agent Skills

Official [Agent Skills](https://agentskills.io) for the
[r4m.ai](https://r4m.ai) route optimization API — drop route optimization into
your AI agent (Claude Code, Claude Desktop, Cursor, and any tool that supports
the Agent Skills standard).

> This repository is auto-generated from the r4m.ai monorepo. Do not edit it
> directly — changes are overwritten on the next publish.

## Install

```bash
npx skills add r4m-ai/skills
```

This installs every skill in this repository. To add a single skill:

```bash
npx skills add r4m-ai/skills --skill r4m-ai-route-optimization
```

## Skills

| Skill | What it does |
|---|---|
| [`r4m-ai-route-optimization`](./r4m-ai-route-optimization/SKILL.md) | Turn a list of delivery/field-service addresses into the fastest driving order via the r4m.ai API. |

## Skills and the MCP server

The skill is instructions; the [MCP server](https://r4m.ai/docs) is tools.
With both, the agent reads the skill to know *how* to plan a route (addresses
need coordinates, at least three of them, the job is asynchronous) and calls
the MCP tools to *do* it. Without the MCP server the skill falls back to the
SDK or raw HTTP, so it is useful on its own.

## Setup

The skills use the r4m.ai API (via the [`@r4mai/sdk`](https://www.npmjs.com/package/@r4mai/sdk)
client in JS/TS projects, or raw HTTP elsewhere) and need an API key:

- `R4M_API_KEY` — your `acc_...` key (create one at https://r4m.ai).
- `R4M_BASE_URL` — optional, defaults to `https://r4m.ai`.

See the full API reference at https://r4m.ai/docs.

## License

MIT
