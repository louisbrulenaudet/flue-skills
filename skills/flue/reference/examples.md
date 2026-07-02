# Examples

Minimal, production-shaped snippets. Confirm APIs against `pnpm flue docs` for your version. All examples use **Zod** (this project's validator — see `validation-zod.md`).

## Contents

- [Agent with a tool](#agent-with-a-tool)
- [Workflow with a structured result](#workflow-with-a-structured-result)
- [Reusable tool with instance scope](#reusable-tool-with-instance-scope)
- [Business-data access via a tool](#business-data-access-via-a-tool)
- [Routing: Hono app with auth](#routing-hono-app-with-auth)
- [Validation at your own boundary](#validation-at-your-own-boundary)
- [Subagent delegation](#subagent-delegation)
- [Channel ingress with idempotency](#channel-ingress-with-idempotency)

## Agent with a tool

```ts
// src/agents/order-assistant.ts
import { createAgent, type AgentRouteHandler } from '@flue/runtime';
import { lookupOrderStatus } from '../tools/orders.ts';
import { authenticate } from '../lib/auth.ts';

export const description = 'Answers order-status questions for the signed-in customer.';

export const route: AgentRouteHandler = async (c, next) => {
  const user = await authenticate(c.req.header('authorization'));
  if (!user) { return c.json({ error: 'Unauthorized' }, 401); }
  if (user.id !== c.req.param('id')) { return c.notFound(); }
  await next();
};

export default createAgent(() => ({
  model: 'anthropic/claude-haiku-4-5',
  instructions: 'Help customers check order status. Use the tool before answering.',
  tools: [lookupOrderStatus],
}));
```

## Workflow with a structured result

```ts
// src/workflows/triage-ticket.ts
import { createAgent, type FlueContext, type WorkflowRouteHandler } from '@flue/runtime';
import { z } from 'zod';

export const route: WorkflowRouteHandler = async (_c, next) => next();

const triager = createAgent(() => ({
  model: 'anthropic/claude-haiku-4-5',
  instructions: 'Triage the ticket. Return priority and a one-line summary.',
}));

export async function run({ init, log, payload }: FlueContext<{ ticket: string }>) {
  log.info('Triage requested', { length: payload.ticket.length });
  const harness = await init(triager);
  const session = await harness.session();
  const response = await session.prompt(payload.ticket, {
    result: z.object({
      priority: z.enum(['low', 'medium', 'high']),
      summary: z.string(),
    }),
  });
  return response.data;
}
```

Invoke: `curl "http://localhost:3583/workflows/triage-ticket?wait=result" -H 'Content-Type: application/json' -d '{"ticket":"Cannot log in since this morning"}'`

## Reusable tool with instance scope

```ts
// src/tools/orders.ts
import { defineTool } from '@flue/runtime';
import { z } from 'zod';

const statuses = new Map<string, string>();

export const lookupOrderStatus = defineTool({
  name: 'lookup_order_status',
  description: 'Look up the current fulfillment status for one order ID.',
  parameters: z.object({
    orderId: z.string().describe('Order ID like order_1234'),
  }),
  execute: async ({ orderId }) => statuses.get(orderId) ?? 'No order was found.',
});
```

## Business-data access via a tool

Business data lives in your own application database, exposed through a narrow, tenant-scoped tool (`database.md`):

```ts
// src/tools/ticket-tools.ts
import { defineTool } from '@flue/runtime';
import { z } from 'zod';

export function createTicketTools(db: TicketStore, tenantId: string) {
  return [
    defineTool({
      name: 'get_ticket',
      description: 'Fetch one ticket by id for the current tenant.',
      parameters: z.object({ ticketId: z.string() }),
      execute: async ({ ticketId }) => {
        const row = await db.getById(ticketId, tenantId); // tenantId bound in code
        return row ? JSON.stringify(row) : 'No ticket found.';
      },
    }),
  ];
}
```

## Routing: Hono app with auth

```ts
// src/app.ts
import { flue } from '@flue/runtime/routing';
import { observe } from '@flue/runtime';
import { Hono, type MiddlewareHandler } from 'hono';
import { authenticate } from './lib/auth.ts';

observe((e) => { if (e.type === 'run_end' && e.isError) { console.error('run failed', e.runId, e.error); } });

const requireUser: MiddlewareHandler = async (c, next) => {
  const user = await authenticate(c.req.raw);
  if (!user) { return c.json({ error: 'Unauthorized' }, 401); }
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

## Validation at your own boundary

```ts
import { z } from 'zod';

const TranslateBody = z.object({ text: z.string().min(1), language: z.string().min(1) });
export type TranslateBody = z.infer<typeof TranslateBody>;

app.post('/translate', async (c) => {
  const parsed = TranslateBody.safeParse(await c.req.json());
  if (!parsed.success) { return c.json({ error: parsed.error.issues }, 400); }
  // dispatch / invoke a workflow with parsed.data ...
});
```

## Subagent delegation

```ts
// src/agents/policy-assistant.ts
import { createAgent, defineAgentProfile } from '@flue/runtime';

const policyResearcher = defineAgentProfile({
  name: 'policy_researcher',
  description: 'Finds relevant policy text and quotes supporting passages.',
  instructions: 'Read the policy workspace and return supporting quotations with file paths.',
});

export default createAgent(() => ({
  model: 'anthropic/claude-sonnet-4-6',
  instructions: 'Answer policy questions. Delegate research, then verify quotes before citing.',
  subagents: [policyResearcher],
}));
```

## Channel ingress with idempotency

```ts
// src/app.ts (excerpt)
import { dispatch } from '@flue/runtime';
import assistant from './agents/support-assistant.ts';
import { db } from './db/index.ts';

app.post('/webhooks/support-comments', async (c) => {
  const event = await parseVerifiedSupportComment(c.req.raw); // signature verified here
  const firstSeen = await db.claimOnce(event.id);             // your durable dedup store
  if (!firstSeen) { return c.json({ ok: true, duplicate: true }, 202); }

  const receipt = await dispatch(assistant, {
    id: event.ticketId,
    input: { type: 'support.comment.created', text: event.text },
  });
  return c.json(receipt, 202);
});
```
