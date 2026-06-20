# Tools

## Contents

- [What a tool is](#what-a-tool-is)
- [defineTool](#definetool)
- [Attaching tools](#attaching-tools)
- [Designing safe, narrow tools](#designing-safe-narrow-tools)
- [Scoping access by instance](#scoping-access-by-instance)
- [Tools vs sandbox vs subagent vs workflow](#tools-vs-sandbox-vs-subagent-vs-workflow)
- [Organization](#organization)
- [Anti-patterns](#anti-patterns)

## What a tool is

A tool lets the model **call your application code** while it works — look up an order, create a ticket, query a database, call an API. This is distinct from a skill, which provides reusable *instructions* (`skills.md`). File and shell access inside the workspace come from the agent's **sandbox**, not from custom tools — don't reimplement those as tools.

## defineTool

```ts
import { defineTool } from '@flue/runtime';
import { z } from 'zod';

export const lookupOrderStatus = defineTool({
  name: 'lookup_order_status',
  description: 'Look up the current fulfillment status for one order ID.',
  parameters: z.object({
    orderId: z.string().describe('Order ID in the form order_1234'),
  }),
  execute: async ({ orderId }) => {
    const status = orderStatuses.get(orderId);
    return status ?? 'No order was found.';
  },
});
```

Fields:

| Field         | Type                                              | Notes                                                            |
| ------------- | ------------------------------------------------- | ---------------------------------------------------------------- |
| `name`        | `string`                                          | Model-facing tool name (snake_case reads well to models).        |
| `description` | `string`                                          | Helps the model decide when the tool applies — be specific.      |
| `parameters`  | Zod `z.object({...})` **or raw JSON Schema**       | Inputs; validated against this schema before `execute` runs.     |
| `execute`     | `async (args) => string`                          | Performs the work; returns text for the model's response.        |

Arguments are validated against `parameters` before `execute` is called.

> **Schema note:** the Flue docs show valibot for `parameters`, but raw JSON Schema is explicitly accepted and Zod implements Standard Schema. Pass the Zod schema directly if your version accepts it, otherwise `z.toJSONSchema(schema)`. See `validation-zod.md`.

## Attaching tools

In agent/profile config via the `tools` array, or per-operation:

```ts
export default createAgent(() => ({
  model: 'anthropic/claude-haiku-4-5',
  instructions: 'Help customers check order status.',
  tools: [lookupOrderStatus],
}));
```

Tools can also be supplied per operation in `session.prompt()`, `session.skill()`, or `session.task()` options.

## Designing safe, narrow tools

- **One capability per tool.** `lookup_order_status` not `manage_orders`. Narrow tools are easier for the model to use correctly and easier to authorize.
- **Validate every input** in `parameters`; add `.describe(...)` so the model supplies well-formed values.
- **Return concise, model-readable text.** Summarize; don't dump raw rows. Return a clear "not found"/error string rather than throwing where the model should recover.
- **Bind authority in code, not in arguments.** The tool — not the model — holds credentials, tenant id, and routing context. Never accept an API key, tenant id, or destination as a model-supplied parameter (see `channels.md` for the message-tool pattern).
- **Least privilege.** Give a tool only the binding/scope it needs.

## Scoping access by instance

Build tools per instance so each can only touch its own resources:

```ts
export default createAgent(({ id }) => ({
  model: 'anthropic/claude-haiku-4-5',
  tools: createTicketTools(id), // closure binds id; tools can't reach other tickets
}));
```

## Tools vs sandbox vs subagent vs workflow

| Need                                                | Use            |
| --------------------------------------------------- | -------------- |
| Read/write files or run commands in the workspace   | the **sandbox**|
| A single discrete action against your code/API      | a **tool**     |
| A multi-step specialist with its own reasoning      | a **subagent** |
| A finite end-to-end job admitted over HTTP          | a **workflow** |

If a "tool" is growing its own multi-step reasoning, promote it to a subagent or workflow.

## Organization

Keep tools in a non-discovered folder (e.g. `src/tools/`) and import them. Group related tools per domain and export factory functions (`createTicketTools(id)`) when they need instance scope.

## Anti-patterns

- Accepting secrets/tenant ids as model parameters.
- One mega-tool with a mode/action enum instead of several narrow tools.
- Throwing on expected "not found" cases the model should handle.
- Reimplementing sandbox file/shell access as custom tools.
