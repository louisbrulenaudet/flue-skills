# Recipe: add a channel (verified provider ingress)

**Purpose.** Accept verified external events (e.g. Teams, GitHub) and route them into agents via `dispatch()`.

**Inputs.** Provider name; provider app credentials/secrets; the agent that handles events; a dedup store.
**Outputs.** A channel module under `channels/` (scaffolded by `flue add`), env secrets, and idempotent dispatch.
**Success criteria.** Verified events reach the agent; invalid signatures are rejected; duplicate deliveries don't double-process; the endpoint returns 202 fast.

## Steps

1. **Confirm the API:** `pnpm flue docs` ("channels"). See `../reference/channels.md`.
2. **Scaffold:**
   ```bash
   flue add channel teams   # installs @flue/teams, lib/teams-client.ts, channels/teams.ts
   ```
   Review the generated files — you own them now.
3. **Set secrets** (provider-specific), e.g. `TEAMS_APP_ID`, `TEAMS_TENANT_ID`, `TEAMS_APP_PASSWORD`:
   ```bash
   pnpm wrangler secret put TEAMS_APP_ID   # repeat per secret
   ```
4. **Add a deduplication store** (`../reference/database.md`) and claim activity ids before dispatch — the framework does not deduplicate and providers retry.
5. **Bind the outbound message tool** to the handling agent if it must reply:
   ```ts
   tools: [postMessage(channel.parseConversationKey(id))],
   ```
6. **Point the provider webhook** at `https://<host>/channels/<name>/<suffix>` (Teams: `/channels/teams/activities`).

## Verification

```bash
pnpm flue build --target cloudflare
pnpm flue dev --target cloudflare
# Send a signed test activity from the provider's tooling; confirm:
#  - valid event -> agent runs (check `flue logs <runId>`)
#  - tampered/invalid signature -> rejected
#  - same activity id twice -> processed once (dedup table)
#  - endpoint returns 202 quickly (does not block on the agent)
```

## References
`../reference/channels.md`, `../reference/database.md`, `../reference/security.md`, `../reference/agents.md`
