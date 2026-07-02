# Deployment: Cloudflare Workers

## Contents

- [Install and scaffold](#install-and-scaffold)
- [wrangler.jsonc](#wranglerjsonc)
- [Durable Object migrations (required)](#durable-object-migrations-required)
- [Build and deploy](#build-and-deploy)
- [Secrets](#secrets)
- [Local development](#local-development)
- [Static assets](#static-assets)
- [Deployment verification](#deployment-verification)
- [Operational cautions](#operational-cautions)

## Install and scaffold

```bash
mkdir my-flue-worker && cd my-flue-worker
npm init -y
npm install @flue/runtime zod 'agents@^0.14.1'
npm install -D @flue/cli wrangler
pnpm flue init --target cloudflare
```

Cloudflare requires the Agents SDK (`agents@^0.14.1`) — it backs the Durable Objects Flue uses for durable runs and sessions.

## wrangler.jsonc

Author this at the project root:

```jsonc
{
  "$schema": "./node_modules/wrangler/config-schema.json",
  "name": "my-flue-worker",
  "compatibility_date": "2026-06-01",
  "compatibility_flags": ["nodejs_compat"],
  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["FlueRegistry", "FlueTranslateWorkflow"] }
  ]
}
```

- `compatibility_date`: `2026-06-01` or later.
- `compatibility_flags`: must include `nodejs_compat`.
- Add any business-data and platform bindings here too (your own database binding, KV, AI, vars) — see `database.md`. These are your application's; they are separate from Flue's own DO SQLite store.

## Durable Object migrations (required)

Every agent and workflow generates a Durable Object class that must be listed in a uniquely-tagged migration using `new_sqlite_classes`. Naming derives from the file name:

| Source file                       | DO class                 | Binding                       |
| --------------------------------- | ------------------------ | ----------------------------- |
| `.flue/workflows/translate.ts`    | `FlueTranslateWorkflow`  | `FLUE_TRANSLATE_WORKFLOW`     |
| `.flue/agents/support-chat.ts`    | `FlueSupportChatAgent`   | `FLUE_SUPPORT_CHAT_AGENT`     |

Always include `FlueRegistry`. When you add agents/workflows later, **append a new tag** (never rewrite an existing one):

```jsonc
{
  "migrations": [
    { "tag": "v1", "new_sqlite_classes": ["FlueRegistry", "FlueTranslateWorkflow"] },
    { "tag": "v2", "new_sqlite_classes": ["FlueAssistantAgent"] }
  ]
}
```

> Use `new_sqlite_classes`, **not** the legacy `new_classes`. A KV-backed Durable Object cannot convert to SQLite in place.

## Build and deploy

```bash
pnpm flue build --target cloudflare
pnpm wrangler deploy --dry-run --config dist/my-flue-worker/wrangler.json   # verify first
pnpm wrangler deploy --config dist/my-flue-worker/wrangler.json
```

> **Deploy the generated config**, `dist/<name>/wrangler.json` — not your source `wrangler.jsonc`. The generated config carries Flue's entrypoint and DO bindings.

## Secrets

Local (`.dev.vars`, gitignored):

```bash
cat > .dev.vars <<'EOF'
ANTHROPIC_API_KEY="your-api-key"
EOF
printf '\n.dev.vars*\n.env*\n' >> .gitignore
```

Production:

```bash
pnpm wrangler secret put ANTHROPIC_API_KEY
pnpm flue build --target cloudflare
pnpm wrangler deploy --config dist/my-flue-worker/wrangler.json
```

In CI, `wrangler deploy --secrets-file <path>` sets secrets non-interactively. Provider keys follow the model you use (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `CLOUDFLARE_API_TOKEN`; the `cloudflare/...` provider uses the AI binding and needs no key — see `agents.md`).

## Local development

```bash
pnpm flue dev --target cloudflare
```

Builds, watches, and serves a Workers dev environment on **port 3583**. Exercise an endpoint:

```bash
curl "http://localhost:3583/workflows/translate?wait=result" \
  -H "Content-Type: application/json" \
  -d '{"text": "Hello world", "language": "French"}'
```

## Static assets

To serve a front-end, add an `assets` block and route API/Flue prefixes to the Worker first (full detail in `hono-workers.md`):

```jsonc
{
  "assets": {
    "directory": "./dist/client",
    "binding": "ASSETS",
    "not_found_handling": "single-page-application",
    "run_worker_first": ["/api/*", "/_flue/*"]
  }
}
```

## Deployment verification

1. `pnpm flue build --target cloudflare` succeeds.
2. `pnpm wrangler deploy --dry-run --config dist/my-flue-worker/wrangler.json` passes.
3. After deploy, hit a workflow:
   ```bash
   curl "https://my-worker.<subdomain>.workers.dev/workflows/translate?wait=result" \
     -H "Content-Type: application/json" -d '{"text":"Hello world","language":"French"}'
   ```
4. Stream an agent's events:
   ```bash
   curl "https://my-worker.<subdomain>.workers.dev/agents/chat/id-123?offset=-1&live=sse"
   ```
5. Confirm new agents/workflows got a migration tag and a binding.

## Operational cautions

- Forgetting a migration tag for a new agent/workflow ⇒ deploy fails or the class isn't created.
- Deploying the source `wrangler.jsonc` ⇒ wrong/missing entrypoint and bindings.
- Omitting `/_flue/*` from `run_worker_first` when serving assets ⇒ Flue endpoints return `index.html`.
- Rewriting an existing migration tag ⇒ Durable Object migration errors. Append, don't edit.
