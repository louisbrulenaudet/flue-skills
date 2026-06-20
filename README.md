# Flue Skills

Agent Skills pack for building [Flue](https://flueframework.com) AI-agent applications on Cloudflare Workers (Hono + Agents SDK). Available as a plugin for Claude Code, Cursor, and OpenAI Codex.

Load when creating Flue agents, workflows, tools, skills, subagents, channels, routing, persistence, validation, deployment, observability, or tests.

## Install

```bash
npx skills add louisbrulenaudet/flue-skills
```

Then select the skills you wish to install.

New to Flue? Start with the official walkthrough: [flueframework.com/start.md](https://flueframework.com/start.md)

## Available Skill

| Skill | Description |
|---|---|
| [`flue`](./skills/flue) | Entry point for all Flue sub-skills — routes to 16 reference guides and 8 workflow recipes |

See [`skills/flue/README.md`](./skills/flue/README.md) for how agents should use the pack.

## What's inside `skills/flue/`

- **`SKILL.md`** — entrypoint, decision tree, critical rules, and verification gates
- **`reference/`** — one focused file per Flue domain (agents, workflows, tools, channels, deployment, etc.)
- **`workflows/`** — step-by-step recipes with inputs, outputs, success criteria, and verification steps
- **`checks/eval-cases.md`** — representative tasks to validate the pack before relying on it

### Sub-skills (reference topics)

| Topic | File |
|---|---|
| Project layout, `src/` vs `.flue/`, naming | [`reference/project-layout.md`](./skills/flue/reference/project-layout.md) |
| HTTP surface, middleware, mounting `flue()` | [`reference/routing.md`](./skills/flue/reference/routing.md) |
| Hono + Workers entrypoint, assets, request flow | [`reference/hono-workers.md`](./skills/flue/reference/hono-workers.md) |
| `createAgent`, profiles, sessions, HTTP exposure | [`reference/agents.md`](./skills/flue/reference/agents.md) |
| `run()`, `FlueContext`, sessions, result schemas | [`reference/workflows.md`](./skills/flue/reference/workflows.md) |
| `defineTool`, parameters, safe interfaces | [`reference/tools.md`](./skills/flue/reference/tools.md) |
| Flue skills (`SKILL.md`, `with { type: 'skill' }`) | [`reference/skills.md`](./skills/flue/reference/skills.md) |
| Subagent profiles, delegation boundaries | [`reference/subagents.md`](./skills/flue/reference/subagents.md) |
| Verified provider ingress, dispatch, idempotency | [`reference/channels.md`](./skills/flue/reference/channels.md) |
| Zod validation, type inference, schema slots | [`reference/validation-zod.md`](./skills/flue/reference/validation-zod.md) |
| Flue persistence + your own business database | [`reference/database.md`](./skills/flue/reference/database.md) |
| `flue build`, wrangler, migrations, secrets, deploy | [`reference/deployment-cloudflare.md`](./skills/flue/reference/deployment-cloudflare.md) |
| `ctx.log`, `observe()`, OTel/Braintrust/Sentry | [`reference/observability.md`](./skills/flue/reference/observability.md) |
| vitest-evals, local dev, pre-deploy checks | [`reference/testing.md`](./skills/flue/reference/testing.md) |
| Auth boundaries, secrets, prompt injection, isolation | [`reference/security.md`](./skills/flue/reference/security.md) |
| End-to-end copy-pasteable examples | [`reference/examples.md`](./skills/flue/reference/examples.md) |

## Plugins

This repo serves as a plugin for multiple platforms:

- **Claude Code** — [`.claude-plugin/`](./.claude-plugin/)
- **Cursor** — [`.cursor-plugin/`](./.cursor-plugin/)
- **OpenAI Codex** — [`.codex-plugin/`](./.codex-plugin/)

## Editing skills

All content is authored in [`skills/flue/`](./skills/flue/). Edit directly in this repo.

Before changing API guidance, confirm against the installed Flue version via `npx flue docs` or [flueframework.com/docs](https://flueframework.com/docs).

## Prerequisites

- Node.js 20+
- A Flue project with the `npx flue` CLI
- Cloudflare account (optional — for Workers deployment; Flue also runs on Node.js)
- Confirm APIs via `npx flue docs` or [flueframework.com/docs](https://flueframework.com/docs)

## Links

- [Flue — The Open Agent Framework](https://flueframework.com)
- [Getting started walkthrough](https://flueframework.com/start.md)
- [Documentation](https://flueframework.com/docs)
