# Validation with Zod

## Contents

- [Project standard: Zod](#project-standard-zod)
- [Where schemas appear in Flue](#where-schemas-appear-in-flue)
- [Zod in Flue tool parameters and result schemas](#zod-in-flue-tool-parameters-and-result-schemas)
- [Native type inference](#native-type-inference)
- [Schema co-location](#schema-co-location)
- [Safe defaults and error shaping](#safe-defaults-and-error-shaping)
- [Examples](#examples)
- [Anti-patterns](#anti-patterns)

## Project standard: Zod

This project uses **Zod (v4)** as its single runtime validator. All examples in this pack use Zod.

> **One compatibility note.** Flue's *official* documentation examples use valibot for tool `parameters` and structured `result` schemas. Both Zod v4 and valibot implement the **Standard Schema** interface, and `defineTool.parameters` is documented to accept a schema object **or raw JSON Schema**. So Zod fits both slots — pass the Zod schema directly if your Flue version accepts a Standard Schema validator, otherwise pass `z.toJSONSchema(schema)`. **Confirm against `pnpm flue docs` for your installed version** and prefer the direct-schema form when it works.

## Where schemas appear in Flue

1. **Tool inputs** — `defineTool({ parameters })`, validated before `execute` runs (`tools.md`).
2. **Structured agent/workflow results** — `session.prompt(text, { result })` and `session.skill(name, { result })`; the model must return data satisfying the schema (`workflows.md`, `skills.md`).
3. **Your own HTTP boundaries** — request bodies in `src/app.ts` Hono routes and webhook payloads in channels. Plain code you control — use Zod with `safeParse`.

## Zod in Flue tool parameters and result schemas

```ts
import { defineTool } from '@flue/runtime';
import { z } from 'zod';

export const lookupOrderStatus = defineTool({
  name: 'lookup_order_status',
  description: 'Look up the current fulfillment status for one order ID.',
  parameters: z.object({
    orderId: z.string().describe('Order ID in the form order_1234'),
  }),
  // If your Flue version does not accept a Zod schema here directly, use:
  //   parameters: z.toJSONSchema(z.object({ orderId: z.string() })),
  execute: async ({ orderId }) => orderStatuses.get(orderId) ?? 'No order was found.',
});
```

Structured result:

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

## Native type inference

Infer types from the schema; never hand-maintain a parallel `type`.

```ts
import { z } from 'zod';

export const TranslateInputSchema = z.object({ text: z.string(), language: z.string() });
export type TranslateInput = z.infer<typeof TranslateInputSchema>;
```

The inferred type feeds typed workflow payloads:

```ts
export async function run({ payload }: FlueContext<TranslateInput>) { /* payload is typed */ }
```

## Schema co-location

Keep a schema next to the code that owns the shape: tool schemas beside the tool, result schemas beside the workflow, request schemas beside the route. Export `SomethingSchema` plus the inferred `type Something` (no `Type` suffix). Don't duplicate a shape — import the schema.

## Safe defaults and error shaping

- Parse with `safeParse` at trust boundaries and return a structured `400` rather than throwing.
- Add `.describe(...)` to fields so models produce well-formed tool inputs.
- Keep error responses generic to callers; log detailed issues server-side (`observability.md`, `security.md`).

## Examples

Request + response + inferred types at your own boundary:

```ts
import { z } from 'zod';

export const SummarizeRequestSchema = z.object({ text: z.string().min(1) });
export const SummarizeResponseSchema = z.object({ summary: z.string() });

export type SummarizeRequest = z.infer<typeof SummarizeRequestSchema>;
export type SummarizeResponse = z.infer<typeof SummarizeResponseSchema>;

app.post('/summarize', async (c) => {
  const parsed = SummarizeRequestSchema.safeParse(await c.req.json());
  if (!parsed.success) { return c.json({ error: parsed.error.issues }, 400); }
  // ... invoke/dispatch with parsed.data ...
});
```

## Anti-patterns

- Maintaining a hand-written `type` alongside a schema (drift) — infer instead.
- Using `.parse()` (throwing) at an external boundary where a `400` is the right response.
- Assuming Zod is wired into every Flue slot without checking — verify, and fall back to `z.toJSONSchema()` for `parameters` if needed.
- A `Type` suffix on inferred types — name them without `Schema`.
