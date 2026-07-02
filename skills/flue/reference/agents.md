# Agents

## Contents

- [What an agent is](#what-an-agent-is)
- [createAgent](#createagent)
- [The three module exports](#the-three-module-exports)
- [Markdown instructions and skills](#markdown-instructions-and-skills)
- [Profiles](#profiles)
- [Subagents](#subagents)
- [Agent ids and sessions](#agent-ids-and-sessions)
- [HTTP exposure and dispatch](#http-exposure-and-dispatch)
- [Model selection](#model-selection)
- [Ephemeral vs durable state](#ephemeral-vs-durable-state)
- [Anti-patterns](#anti-patterns)

## What an agent is

An agent is a **continuing instance** that accepts messages over time. Use an agent when conversation continuity matters (a support thread, an assistant tied to a ticket). For a finite one-shot job, use a workflow instead (`workflows.md`).

Define one agent per file in `src/agents/`; the **filename is the agent name** (`support-assistant.ts` → `support-assistant`).

## createAgent

The default export is `createAgent(factory)`. The factory receives `{ id }` (the instance id) and returns config:

```ts
import { createAgent } from '@flue/runtime';

export default createAgent(({ id }) => ({
  model: 'anthropic/claude-sonnet-4-6',
  instructions: 'Answer customer support questions clearly and accurately.',
  // optional:
  // cwd, tools, skills, sandbox, profile, subagents, thinkingLevel
}));
```

Configuration fields (confirm against `pnpm flue docs` for your version):

| Field          | Purpose                                                        |
| -------------- | ------------------------------------------------------------- |
| `model`        | `provider/model` string (see `model selection`).              |
| `instructions` | Behavior guidance (string or imported markdown).              |
| `cwd`          | Working directory for the sandbox workspace.                  |
| `tools`        | Array of `defineTool(...)` tools (see `tools.md`).            |
| `skills`       | Array of imported skills (see `skills.md`).                   |
| `sandbox`      | Execution environment config.                                 |
| `profile`      | A reusable `defineAgentProfile(...)` preset.                  |
| `subagents`    | Array of profiles the agent may delegate to.                  |
| `thinkingLevel`| `'off' | 'minimal' | 'low' | 'medium' | 'high' | 'xhigh'`.    |

## The three module exports

```ts
import { createAgent, type AgentRouteHandler } from '@flue/runtime';

// 1) Optional static metadata collected at build time.
export const description = 'Tells a short joke in response to each message.';

// 2) Optional HTTP guard. Without it, the agent is NOT reachable over HTTP.
export const route: AgentRouteHandler = async (_c, next) => next();

// 3) Required default export: behavior + environment.
export default createAgent(() => ({
  model: 'anthropic/claude-haiku-4-5',
  instructions: 'Tell a short joke in response to each message.',
}));
```

## Markdown instructions and skills

Long instructions import from markdown with a required import attribute:

```ts
import instructions from './repository-reviewer.md' with { type: 'markdown' };

export default createAgent(() => ({
  model: 'anthropic/claude-sonnet-4-6',
  instructions,
}));
```

Skills use `with { type: 'skill' }` and go in the `skills` array — see `skills.md`.

## Profiles

`defineAgentProfile(...)` bundles a reusable configuration (model + instructions + tools) without creating a public endpoint. Apply it via `profile`, and override per instance:

```ts
import { createAgent, defineAgentProfile } from '@flue/runtime';
import { supportTools } from '../tools/support.ts';

const support = defineAgentProfile({
  model: 'anthropic/claude-haiku-4-5',
  instructions: 'Answer customer support questions clearly and accurately.',
  tools: supportTools,
});

export default createAgent(() => ({ profile: support }));
```

## Subagents

Subagents are profiles (with `name` + `description`) the parent delegates to. They have **no public endpoint** and are callable only by the parent. See `subagents.md`.

```ts
const policyResearcher = defineAgentProfile({
  name: 'policy_researcher',
  description: 'Finds relevant policy text and quotes supporting passages.',
  instructions: 'Read the policy workspace and return supporting quotations with file paths.',
});

export default createAgent(() => ({
  model: 'anthropic/claude-sonnet-4-6',
  subagents: [policyResearcher],
}));
```

## Agent ids and sessions

Each instance has an `id` that identifies the continuing session, supplied in the route: `POST /agents/support-assistant/ticket-8472` → `id = "ticket-8472"`. You choose the semantics (user, ticket, issue, random). Use the id to scope resources:

```ts
export default createAgent(({ id }) => ({
  model: 'anthropic/claude-haiku-4-5',
  tools: createTicketTools(id), // scoped to this instance
}));
```

## HTTP exposure and dispatch

- **Synchronous prompt** — `POST /agents/:name/:id` with a body:
  ```json
  {
    "message": "Can you summarize the open issues?",
    "images": [{ "type": "image", "data": "<base64>", "mimeType": "image/png" }]
  }
  ```
  Requires a `route` export. Stream events with `GET /agents/:name/:id`.

- **Asynchronous dispatch** — queue work without waiting; needs no public transport:
  ```ts
  import { dispatch } from '@flue/runtime';

  const receipt = await dispatch(supportAssistant, {
    id: event.ticketId,
    input: { type: 'support.comment.created', commentId: event.commentId, text: event.text },
  });
  ```

Put coarse auth in route middleware (`routing.md`) and per-instance ownership checks in the `route` export:

```ts
export const route: AgentRouteHandler = async (c, next) => {
  const principal = await authenticate(c.req.header('authorization'));
  const ticketId = c.req.param('id');
  if (!principal) { return c.json({ error: 'Unauthorized' }, 401); }
  if (!principal.supportTicketIds.includes(ticketId)) { return c.notFound(); }
  await next();
};
```

## Model selection

Models are `provider/model` strings. Defaults and providers:

- `anthropic/claude-sonnet-4-6`, `anthropic/claude-haiku-4-5` (`ANTHROPIC_API_KEY`)
- `openai/gpt-5.5` (`OPENAI_API_KEY`)
- `openrouter/<vendor>/<model>` (`OPENROUTER_API_KEY`)
- `cloudflare/@cf/<vendor>/<model>` — Workers AI binding, no API key
- `cloudflare-workers-ai/...`, `cloudflare-ai-gateway/...` (`CLOUDFLARE_API_TOKEN`)

Custom providers register in `src/app.ts` via `registerProvider('ollama', { api: 'openai-completions', baseUrl: '...' })`, then `model: 'ollama/llama3.1:8b'`. Choose the smallest model that meets the quality bar; reserve Sonnet/`thinkingLevel: 'high'` for hard reasoning.

## Ephemeral vs durable state

Flue persists session snapshots and messages (DO SQLite on Cloudflare). That is **not** your business database. Keep customer records, tickets, and payments in your own store (`database.md`), and treat the sandbox workspace as ephemeral — a persisted session does not make a sandbox durable.

## Anti-patterns

- Forgetting the `route` export and wondering why the agent 404s over HTTP.
- Using an agent for a finite job that should be a workflow.
- Putting business data in session state instead of your own database.
- Hard-coding ids; let the caller/route define id semantics.
