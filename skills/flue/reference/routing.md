# Routing

## Contents

- [The HTTP surface](#the-http-surface)
- [src/app.ts — your own Hono app](#srcappts--your-own-hono-app)
- [Middleware and auth boundaries](#middleware-and-auth-boundaries)
- [Mounting under a prefix](#mounting-under-a-prefix)
- [Custom routes and dispatch](#custom-routes-and-dispatch)
- [Exposure rules](#exposure-rules)
- [Anti-patterns](#anti-patterns)

## The HTTP surface

When `flue()` is mounted it exposes a fixed surface:

- **Agents** — `POST /agents/:name/:id` (prompt), `GET /agents/:name/:id` (event stream).
- **Workflows** — `POST /workflows/:name` (invoke). Returns `202 { runId, streamUrl, offset }`; add `?wait=result` to wait synchronously.
- **Channels** — provider-declared surfaces under `/channels/:name/<suffix>`.
- **Run reads** — `GET /runs/:runId` (event stream and `?meta` metadata).

These routes are not gated by any module export, except that a workflow's optional `route` middleware runs for its own endpoint. Protect the whole surface with application middleware (below).

## src/app.ts — your own Hono app

`src/app.ts` is the optional entrypoint for supplying your own HTTP application. Import `flue()` from `@flue/runtime/routing` and mount it on a Hono instance:

```ts
// src/app.ts
import { flue } from '@flue/runtime/routing';
import { Hono, type MiddlewareHandler } from 'hono';
import { authenticate } from './lib/auth.ts';

const requireUser: MiddlewareHandler = async (c, next) => {
  const user = await authenticate(c.req.raw);
  if (!user) {
    return c.json({ error: 'Unauthorized' }, 401);
  }
  await next();
};

const app = new Hono();
app.get('/health', (c) => c.json({ ok: true }));
app.use('/agents/*', requireUser);
app.use('/workflows/*', requireUser);
app.use('/runs/*', requireUser);
app.use('/channels/*', requireUser);
app.route('/', flue());

export default app;
```

Without `src/app.ts`, Flue mounts its routes at `/` automatically with no application middleware — acceptable only for trusted/local use.

## Middleware and auth boundaries

Apply broad middleware (authentication, rate limiting) above the mounted `flue()` app so it covers every run regardless of workflow. Apply resource-specific checks (does this principal own this ticket id?) inside the relevant agent/workflow `route` export. See `agents.md` and `security.md`.

Run reads are sensitive: an unknown run and an unauthorized run should return the **same `404` shape**, never `401`/`403`, so middleware does not disclose run existence.

## Mounting under a prefix

Mount `flue()` beneath a path prefix to coexist with a larger API:

```ts
const app = new Hono();
app.get('/health', (c) => c.json({ ok: true }));
app.route('/api', flue());
export default app;
```

Agents are then at `/api/agents/:name/:id`, workflows at `/api/workflows/:name`. SDK clients must include the prefix: `createFlueClient({ baseUrl: 'https://example.com/api' })`.

## Custom routes and dispatch

Application-owned routes (e.g. webhooks) decide which requests are valid and which agent instance receives input, then hand off with `dispatch()`:

```ts
import { dispatch } from '@flue/runtime';
import { flue } from '@flue/runtime/routing';
import { Hono } from 'hono';
import supportAssistant from './agents/support-assistant.ts';

const app = new Hono();
app.post('/webhooks/support-comments', async (c) => {
  const event = await parseVerifiedSupportComment(c.req.raw);
  const receipt = await dispatch(supportAssistant, {
    id: event.ticketId,
    input: { type: 'support.comment.created', text: event.text },
  });
  return c.json(receipt, 202);
});
app.route('/', flue());
export default app;
```

`dispatch()` queues durable work and returns a receipt immediately — the right shape for webhooks and event ingress (see `channels.md`).

## Exposure rules

- An **agent** is invocable over HTTP only if it exports a `route` (`AgentRouteHandler`). Agents used only via `dispatch()` need no public transport.
- A **workflow** is invocable over HTTP only if it exports a `route` (`WorkflowRouteHandler`).
- **Channels** publish automatically per their provider declaration.

## Anti-patterns

- Relying on the implicit `/` mount in production with no middleware.
- Returning `401`/`403` on run reads (discloses existence) — return `404`.
- Forgetting to update the SDK `baseUrl` after mounting under `/api`.
- Putting per-resource authorization in global middleware where it can't see the resource owner — push it into the `route` export.
