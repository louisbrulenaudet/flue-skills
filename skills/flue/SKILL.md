---
name: flue
description: >-
  Build Flue AI-agent applications on Cloudflare Workers with Hono and the
  Agents SDK. Load when creating Flue agents, workflows, tools, skills,
  subagents, channels, routing, persistence, validation, deployment,
  observability, or tests. Entry point for all Flue sub-skills.
type: core
library: flue
library_version: '1.x (1.0 Beta line)'
license: MIT
---

# Flue

Flue is a TypeScript framework for building AI agents using the harness-driven architecture popularized by coding agents. An application is a set of **agents** (continuing instances that accept messages over time), **workflows** (finite agent-backed operations), **tools** (application code the model can call), **skills** (reusable instructions), and **channels** (verified provider ingress). Flue mounts a Hono HTTP surface and runs on Node.js or Cloudflare Workers; on Cloudflare it uses the Agents SDK and Durable Object SQLite for durable runs and sessions.

> **CRITICAL — documentation first.** Flue's API surface is young and version-sensitive. Before writing or editing Flue code, confirm the exact API against the installed version: run `pnpm flue docs` (bundled offline docs) or read https://flueframework.com/docs. Do not invent APIs, bindings, file layouts, or runtime behavior. If something is not in the docs, omit it or mark it as a recommendation.

> **Package manager — pnpm.** This skill pack assumes **pnpm** to run project-local CLIs: `pnpm flue …`, `pnpm wrangler …`. Do not use `npx` for Flue or Wrangler commands. Ensure `flue` and `wrangler` are in your project's `devDependencies` (or `dependencies`) so pnpm resolves them. The one-time skill-pack install (`npx skills add …`) is the only exception.

> **CRITICAL — schema library.** This project standardizes on **Zod (v4)** for all validation, see `reference/validation-zod.md` for the exact pattern and the `z.toJSONSchema()` fallback.

> **CRITICAL — Flue manages its own persistence.** Flue stores session/run state in Durable Object SQLite automatically on Cloudflare (or via a `db.ts` adapter on Node) — never wire an external database in as Flue's session store. Keep your **business data** in your own application database. See `reference/database.md`.

## Sub-skills

| Task                                                      | File                                              |
| --------------------------------------------------------- | ------------------------------------------------- |
| Where files go, `src/` vs `.flue/`, naming                | [reference/project-layout.md](./reference/project-layout.md) |
| HTTP surface, `src/app.ts`, middleware, mounting `flue()` | [reference/routing.md](./reference/routing.md)    |
| Hono + Workers entrypoint, assets, request flow           | [reference/hono-workers.md](./reference/hono-workers.md) |
| `createAgent`, profiles, ids, sessions, HTTP exposure     | [reference/agents.md](./reference/agents.md)      |
| `run()`, `FlueContext`, sessions, result schemas          | [reference/workflows.md](./reference/workflows.md) |
| `defineTool`, parameters, safe interfaces                 | [reference/tools.md](./reference/tools.md)        |
| Flue skills (`SKILL.md`, `with { type: 'skill' }`)        | [reference/skills.md](./reference/skills.md)      |
| Subagent profiles, delegation boundaries                  | [reference/subagents.md](./reference/subagents.md) |
| Verified provider ingress, dispatch, idempotency          | [reference/channels.md](./reference/channels.md)  |
| Zod validation, type inference, Flue schema slots         | [reference/validation-zod.md](./reference/validation-zod.md) |
| Flue persistence + your own business database             | [reference/database.md](./reference/database.md)  |
| `flue build`, wrangler, migrations, secrets, deploy       | [reference/deployment-cloudflare.md](./reference/deployment-cloudflare.md) |
| `ctx.log`, `observe()`, OTel/Braintrust/Sentry            | [reference/observability.md](./reference/observability.md) |
| vitest-evals, local dev, pre-deploy checks                | [reference/testing.md](./reference/testing.md)    |
| Auth boundaries, secrets, prompt injection, isolation     | [reference/security.md](./reference/security.md)  |
| End-to-end copy-pasteable examples                        | [reference/examples.md](./reference/examples.md)  |

Step-by-step task recipes live in [`workflows/`](./workflows/). Start there for a concrete job; drop into `reference/` for depth.

## Quick decision tree

```
Need a continuing, conversational instance that accepts messages over time?  → agents
Need a finite, one-shot agent-backed job (summarize, review, transform)?      → workflows
Need the model to call your code (DB lookup, API call, action)?               → tools
Need reusable instructions/process the agent loads on demand?                 → skills
Need a specialist the parent agent delegates to (no public endpoint)?         → subagents
Need to accept verified webhooks/events from an external provider?            → channels
Need to validate input/output shapes?                                         → validation-zod
Need durable business data (customers, tickets, payments)?                    → database
Shipping to Cloudflare Workers?                                               → deployment-cloudflare
```

## Default implementation philosophy

Prefer the smallest unit that fits: a **tool** for a discrete action, a **workflow** for a finite job, an **agent** only when conversation continuity matters, a **subagent** for bounded delegation. Validate every trust boundary. Keep Flue's session state separate from durable business data. Expose nothing over HTTP without an explicit `route` export and middleware. Go through Flue's HTTP admission boundary (or `dispatch`) — never call `run()` directly.

## Minimal working example

```ts
// src/agents/hello.ts
import { createAgent } from '@flue/runtime';

export const description = 'Tells a short hello-world joke.';

export default createAgent(() => ({
  model: 'anthropic/claude-haiku-4-5',
  instructions: 'Tell a short "hello world" engineering joke.',
}));
```

```bash
pnpm flue dev                 # watch-mode local server
pnpm flue connect hello demo  # interactive session against instance "demo"
```

## Verification is required before completion

Every workflow recipe in this pack ends with a verification step. Do not report a task done until you have run it. The baseline gate for any Flue change:

```bash
pnpm flue build --target <node|cloudflare>   # must build clean
# then exercise the affected route, e.g.:
curl "http://localhost:3583/workflows/<name>?wait=result" \
  -H 'Content-Type: application/json' -d '{ ... }'
```

## Anti-patterns (do not do)

- Inventing Flue APIs from memory instead of checking `pnpm flue docs` / the live docs.
- Wiring an external database as Flue's session/run store — that is DO SQLite (automatic on Cloudflare) or a `db.ts` adapter (Node).
- Deploying the source `wrangler.jsonc` instead of the generated `dist/<name>/wrangler.json`.
- Adding an agent/workflow without a uniquely-tagged `new_sqlite_classes` migration.
- Calling a workflow's `run()` directly from app code.
- Exposing an agent/workflow without a `route` export and auth middleware.
- One giant skill file or overloaded `CLAUDE.md` — use progressive disclosure.
