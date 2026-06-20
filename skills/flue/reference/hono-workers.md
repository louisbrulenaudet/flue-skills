# Hono + Cloudflare Workers

## Contents

- [How Flue uses Hono](#how-flue-uses-hono)
- [The Workers entrypoint](#the-workers-entrypoint)
- [Request flow and middleware order](#request-flow-and-middleware-order)
- [Static assets](#static-assets)
- [Accessing env and bindings](#accessing-env-and-bindings)
- [Compatibility](#compatibility)
- [Anti-patterns](#anti-patterns)

## How Flue uses Hono

Flue's HTTP surface is a Hono app. `flue()` (from `@flue/runtime/routing`) returns a mountable Hono app exposing the agent/workflow/channel/run routes. You compose it inside your own `Hono` instance in `src/app.ts` (see `routing.md`). On Cloudflare, Flue builds a Worker entrypoint around that app and wires Durable Object bindings for each agent/workflow.

## The Workers entrypoint

For most apps you do **not** hand-write the Worker entrypoint — `flue build --target cloudflare` generates it (and the deployable `dist/<name>/wrangler.json`). Provide custom behavior through:

- `src/app.ts` — your Hono routes and middleware (preferred for HTTP logic).
- `src/cloudflare.ts` — optional custom Cloudflare Worker exports when you need Worker-level control (e.g. additional exported handlers).

The generated entrypoint mounts your `app.ts` if present, otherwise mounts `flue()` at `/`.

## Request flow and middleware order

```
Request
  → asset handler (if assets configured & path not in run_worker_first)
  → Worker entrypoint (generated)
  → your Hono app (src/app.ts): global middleware (auth, rate-limit)
  → flue() routes  → agent/workflow route export → handler
                   → Durable Object (FlueRegistry + per-agent/workflow class)
```

Order matters: global middleware on `/agents/*`, `/workflows/*`, `/runs/*`, `/channels/*` runs **before** the route export. Put coarse auth in middleware, fine-grained ownership checks in the `route` export.

## Static assets

To serve a front-end alongside Flue endpoints, configure the Workers `assets` block and route API prefixes to the Worker first:

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

List **every** API/Flue prefix in `run_worker_first` so those requests reach the Worker before the asset handler intercepts them. Internal Flue paths use the `/_flue/*` prefix — always include it.

## Accessing env and bindings

In Hono handlers and middleware, bindings and vars are on `c.env` (e.g. your business-DB binding, `c.env.ANTHROPIC_API_KEY`). In workflow `run(ctx)`, env is available on `ctx.env`. Pass bindings into tools explicitly rather than reaching for globals — see `tools.md` and `database.md`.

## Compatibility

- `compatibility_date` must be recent enough for the Agents SDK features Flue relies on — use `2026-06-01` or later.
- `compatibility_flags` must include `nodejs_compat`.
- Cloudflare deploys require the Agents SDK: `npm install 'agents@^0.14.1'` alongside `@flue/runtime` (and `zod` for validation).

## Anti-patterns

- Hand-editing the generated Worker entrypoint instead of using `src/app.ts` / `src/cloudflare.ts`.
- Omitting `/_flue/*` (or your API prefix) from `run_worker_first`, so the asset handler swallows API calls and the OAuth/agent flow silently returns `index.html`.
- Reading secrets from module scope at import time instead of from `c.env` per request.
