# Recipe: build and deploy to Cloudflare Workers

**Purpose.** Ship a Flue app to Cloudflare Workers correctly (right config, migrations, secrets).

**Inputs.** A Flue project targeting cloudflare; provider API key(s); any application bindings (your database, KV, AI).
**Outputs.** A deployed Worker; secrets set; migrations applied.
**Success criteria.** Dry-run passes; deploy succeeds; a live workflow/agent request returns the expected response.

## Steps

1. **Confirm config.** `wrangler.jsonc` has `compatibility_date >= 2026-06-01`, `compatibility_flags: ["nodejs_compat"]`, and a `migrations` entry listing `FlueRegistry` plus every `Flue<Name>Agent|Workflow` class. Append a new tag for any new agent/workflow — never edit an existing tag. (`../reference/deployment-cloudflare.md`)
2. **Set secrets:**
   ```bash
   pnpm wrangler secret put ANTHROPIC_API_KEY
   ```
3. **Build:**
   ```bash
   pnpm flue build --target cloudflare
   ```
4. **Dry-run against the generated config:**
   ```bash
   pnpm wrangler deploy --dry-run --config dist/<name>/wrangler.json
   ```
5. **Deploy the generated config** (not the source `wrangler.jsonc`):
   ```bash
   pnpm wrangler deploy --config dist/<name>/wrangler.json
   ```
6. **Apply your application database's migrations** to its production instance if you have one (separate from Flue's own store; see `../reference/database.md`).

## Verification

```bash
curl "https://<name>.<subdomain>.workers.dev/workflows/<wf>?wait=result" \
  -H 'Content-Type: application/json' -d '{ /* payload */ }'
curl "https://<name>.<subdomain>.workers.dev/agents/<agent>/id-123?offset=-1&live=sse"
```
Confirm: response is correct; logs show no missing-binding/migration errors; new classes exist.

## Common failures
- Deployed the source `wrangler.jsonc` instead of `dist/<name>/wrangler.json`.
- Missing migration tag for a new agent/workflow.
- Used legacy `new_classes` instead of `new_sqlite_classes`.
- Assets configured but `/_flue/*` / API prefix missing from `run_worker_first`.

## References
`../reference/deployment-cloudflare.md`, `../reference/hono-workers.md`, `../reference/database.md`
