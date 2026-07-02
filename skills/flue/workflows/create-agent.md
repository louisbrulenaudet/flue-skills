# Recipe: create an agent

**Purpose.** Add a continuing, conversational agent instance exposed (optionally) over HTTP.

**When.** Conversation continuity matters (chat, ticket thread). For a one-shot job use `create-workflow.md`.

**Inputs.** Agent name (kebab-case), model, instructions, optional tools/skills, auth policy.
**Outputs.** `src/agents/<name>.ts` with a default `createAgent` export (and `route` if HTTP-exposed). On Cloudflare, a new migration tag.
**Success criteria.** Builds clean; `flue connect` (or `POST /agents/<name>/<id>`) returns a response; unauthorized callers are rejected.

## Steps

1. **Confirm the API** for your version: `pnpm flue docs` (search "createAgent", "route"). See `../reference/agents.md`.
2. **Create the file** `src/agents/<name>.ts`:
   ```ts
   import { createAgent, type AgentRouteHandler } from '@flue/runtime';

   export const description = '<one-line third-person summary>';

   export const route: AgentRouteHandler = async (c, next) => {
     const user = await authenticate(c.req.header('authorization'));
     if (!user) { return c.json({ error: 'Unauthorized' }, 401); }
     if (!userMayAccess(user, c.req.param('id'))) { return c.notFound(); }
     await next();
   };

   export default createAgent(({ id }) => ({
     model: 'anthropic/claude-haiku-4-5',
     instructions: '<imperative behavior>',
     // tools: createTools(id),  // optional, instance-scoped
   }));
   ```
   Omit `route` only if the agent is reached solely via `dispatch()`.
3. **Attach tools/skills** if needed (`../reference/tools.md`, `../reference/skills.md`).
4. **Cloudflare:** add a migration tag for the new DO class `Flue<Name>Agent` (`../reference/deployment-cloudflare.md`).

## Verification

```bash
pnpm flue build --target <node|cloudflare>     # clean build
pnpm flue dev                                   # in one terminal
pnpm flue connect <name> demo                   # interactive check
# or HTTP:
curl -X POST "http://localhost:3583/agents/<name>/demo" \
  -H 'Authorization: Bearer <token>' -H 'Content-Type: application/json' \
  -d '{"message":"hello"}'
# negative test: same call without auth must 401; wrong id must 404
```

## References
`../reference/agents.md`, `../reference/routing.md`, `../reference/security.md`, `../reference/deployment-cloudflare.md`
