# Workflows

## Contents

- [What a workflow is](#what-a-workflow-is)
- [Structure: run() and FlueContext](#structure-run-and-fluecontext)
- [The harness: sessions, fs, shell](#the-harness-sessions-fs-shell)
- [Typed inputs and structured results](#typed-inputs-and-structured-results)
- [HTTP exposure and run admission](#http-exposure-and-run-admission)
- [Multi-turn and workspace preparation](#multi-turn-and-workspace-preparation)
- [Retries and idempotency](#retries-and-idempotency)
- [Run management](#run-management)
- [Workflow vs agent](#workflow-vs-agent)
- [Anti-patterns](#anti-patterns)

## What a workflow is

A workflow is a **finite, agent-backed operation** — summarize a document, review a change, transform data — with no ongoing conversation. One workflow per file in `src/workflows/`; the **filename is the workflow name** (`summarize.ts` → `summarize`).

## Structure: run() and FlueContext

A workflow exports an async `run(ctx)`:

```ts
import { createAgent, type FlueContext } from '@flue/runtime';

const summarizer = createAgent(() => ({
  model: 'anthropic/claude-haiku-4-5',
  instructions: 'Summarize the supplied document clearly and concisely.',
}));

export async function run({ init, payload }: FlueContext<{ text: string }>) {
  const harness = await init(summarizer);
  const session = await harness.session();
  const response = await session.prompt(payload.text);
  return { summary: response.text };
}
```

`FlueContext<T>` provides:

| Property  | Purpose                                                  |
| --------- | -------------------------------------------------------- |
| `init`    | `init(agent)` → the agent's harness.                     |
| `payload` | Typed input (`T`).                                       |
| `id`      | The run/instance id.                                     |
| `env`     | Bindings and env vars (e.g. your business-DB binding).   |
| `req`     | The inbound request.                                     |
| `log`     | Structured logger (`log.info/warn/error`, see `observability.md`). |

## The harness: sessions, fs, shell

```ts
const harness = await init(agent);
const session = await harness.session();        // default session
await session.prompt('…');                       // send a message
await harness.fs.writeFile('document.md', text); // workspace file write
const out = await harness.fs.readFile('review.md');
await harness.shell('npm test');                 // run a command in the workspace
```

The workspace/sandbox is ephemeral — durable outputs belong in your DB or in the workflow's return value.

## Typed inputs and structured results

Inputs are typed via `FlueContext<T>`. For structured **output**, pass a `result` schema to `prompt`; the agent must return data satisfying it before the workflow proceeds. This project uses **Zod**:

```ts
import { z } from 'zod';

const response = await session.prompt(payload.ticket, {
  result: z.object({
    priority: z.enum(['low', 'medium', 'high']),
    summary: z.string(),
  }),
});
return response.data; // typed
```

> Flue's own docs show valibot here; Zod works via Standard Schema (or `z.toJSONSchema(schema)` as a fallback). See `validation-zod.md`.

## HTTP exposure and run admission

Expose a workflow over HTTP by adding a `route` export:

```ts
import type { WorkflowRouteHandler } from '@flue/runtime';
export const route: WorkflowRouteHandler = async (_c, next) => next();
```

`POST /workflows/:name` returns `202 { runId, streamUrl, offset }` by default; append `?wait=result` to wait synchronously for the result.

> **Always go through the HTTP admission boundary (or `dispatch`).** Do not call `run()` directly from application code — that bypasses durable run storage, events, route middleware, and run-inspection APIs.

## Multi-turn and workspace preparation

A workflow may run several prompts against one session, or stage files first:

```ts
await harness.fs.writeFile('document.md', payload.document);
const session = await harness.session();
await session.prompt('Review document.md and write findings to review.md.');
return { review: await harness.fs.readFile('review.md') };
```

```ts
const session = await harness.session();
await session.prompt(`Analyze this incident:\n\n${payload.incident}`);
const next = await session.prompt('Now recommend the next three actions.');
```

## Retries and idempotency

The Flue workflow docs do **not** document a native retry mechanism, idempotency keys, or a `dispatch`-style queue *inside* a workflow's `run`. Treat these as your responsibility:

- For at-least-once ingress (webhooks/channels), claim the event id in durable storage before doing work, and design `run` to be safe to re-enter. See `channels.md` and `database.md`.
- Do not assume Flue retries a failed workflow for you — verify behavior against `pnpm flue docs` for your version before depending on it.

(`dispatch()` exists for **admitting** async work to an agent/workflow from app code — not as an in-`run` retry primitive.)

## Run management

- `flue logs <runId> --server http://localhost:3583` — replay/follow events.
- `GET /runs/<runId>` — stream events (Durable Streams); `GET /runs/<runId>?meta` — metadata JSON.
- SDK: `client.runs.get()`, `.events()`, `.stream()`; `listRuns()` from a protected server endpoint.

## Workflow vs agent

| Use a workflow                                  | Use an agent                                    |
| ----------------------------------------------- | ----------------------------------------------- |
| Finite job, returns once (summarize, review)    | Continuing instance, many messages over time    |
| Background/batch work                            | Conversational session (ticket, chat)           |
| No conversation state needed                     | Session continuity matters                       |

## Anti-patterns

- Calling `run()` directly instead of using the HTTP boundary / `dispatch`.
- Assuming automatic retries or idempotency — neither is documented.
- Storing durable results only in the sandbox workspace (it's ephemeral).
- Missing the `route` export, then expecting `POST /workflows/:name` to work.
