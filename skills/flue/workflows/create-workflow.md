# Recipe: create a workflow

**Purpose.** Add a finite, agent-backed operation invoked over HTTP, returning a result.

**When.** One-shot job (summarize, triage, transform). For ongoing conversation use `create-agent.md`.

**Inputs.** Workflow name (kebab-case), typed payload shape, optional result schema, auth policy.
**Outputs.** `src/workflows/<name>.ts` exporting `run(ctx)` (and `route` if HTTP-exposed). On Cloudflare, a new migration tag.
**Success criteria.** `POST /workflows/<name>?wait=result` returns the expected typed result; build is clean.

## Steps

1. **Confirm the API:** `pnpm flue docs` ("workflows", "FlueContext", "route"). See `../reference/workflows.md`.
2. **Create** `src/workflows/<name>.ts`:
   ```ts
   import { createAgent, type FlueContext, type WorkflowRouteHandler } from '@flue/runtime';
   import { z } from 'zod';

   export const route: WorkflowRouteHandler = async (_c, next) => next();

   const agent = createAgent(() => ({
     model: 'anthropic/claude-haiku-4-5',
     instructions: '<what the agent should do>',
   }));

   export async function run({ init, log, payload }: FlueContext<{ /* input */ }>) {
     log.info('<name> requested');
     const harness = await init(agent);
     const session = await harness.session();
     const response = await session.prompt(/* build prompt from payload */, {
       result: z.object({ /* structured output */ }),
     });
     return response.data;
   }
   ```
3. **Type the payload** via `FlueContext<T>`; add a `result` schema for structured output (`../reference/validation-zod.md`).
4. **Cloudflare:** add a migration tag for `Flue<Name>Workflow`.

> Do not call `run()` directly from app code — invoke via HTTP or `dispatch()`.

## Verification

```bash
pnpm flue build --target <node|cloudflare>
pnpm flue dev
curl "http://localhost:3583/workflows/<name>?wait=result" \
  -H 'Content-Type: application/json' -d '{ /* payload */ }'
# without ?wait=result you should get 202 { runId, streamUrl, offset }
pnpm flue logs <runId> --server http://localhost:3583   # inspect the run
```

## References
`../reference/workflows.md`, `../reference/validation-zod.md`, `../reference/deployment-cloudflare.md`
