# Security

## Contents

- [Auth boundaries](#auth-boundaries)
- [Least privilege](#least-privilege)
- [Secret handling](#secret-handling)
- [Sandboxing](#sandboxing)
- [Prompt injection and data exfiltration](#prompt-injection-and-data-exfiltration)
- [Tenant isolation](#tenant-isolation)
- [Channel ingress](#channel-ingress)
- [Data retention](#data-retention)
- [Anti-patterns](#anti-patterns)

## Auth boundaries

Two layers (`routing.md`, `agents.md`):

- **Coarse auth in Hono middleware** above `flue()` — authenticate every request to `/agents/*`, `/workflows/*`, `/runs/*`, `/channels/*` before it reaches a handler.
- **Fine-grained ownership in the `route` export** — confirm the authenticated principal may act on *this* instance id (e.g. owns the ticket). Without a `route` export, an agent/workflow is not HTTP-reachable at all.

Return **404 (not 401/403)** for unauthorized run reads so existence isn't disclosed; an unknown run and an unauthorized run must look identical.

## Least privilege

Give each agent, subagent, and tool only what it needs. Scope tools to an instance with a closure over `id`/`tenantId` (`tools.md`) so a compromised or confused agent can't reach another tenant's data. Prefer read-only tools unless a write is required; never hand a research/review subagent write access.

## Secret handling

- Secrets live in `.dev.vars` (local, gitignored) and `wrangler secret put` (production) — never in source or in `wrangler.jsonc` (`deployment-cloudflare.md`).
- Read secrets from `c.env`/`ctx.env` per request, not from module scope at import time.
- **Never** accept secrets, API keys, tenant ids, or routing destinations as model-supplied tool parameters — bind them in trusted code (the channel message-tool pattern, `channels.md`).
- Don't log secrets or full prompts into exported telemetry (`observability.md`).

## Sandboxing

Agent file/shell access comes from the configured sandbox, not custom tools. Treat the sandbox workspace as untrusted and ephemeral: a persisted session does not make a sandbox durable, and nothing the model writes there should be trusted as authoritative. Persist authoritative data in your own database (`database.md`).

## Prompt injection and data exfiltration

Untrusted text (user messages, webhook payloads, fetched documents) can contain instructions aimed at your agent. Mitigations:

- Keep authority in code: the model proposes, tools (with their own checks) dispose. A tool must re-validate every action regardless of why the model called it.
- Don't let tool arguments carry destinations/credentials — the model can be talked into changing an argument, not a code-bound value.
- Constrain outbound tools (which recipients/endpoints are allowed) so an injected instruction can't redirect output.
- Validate structured `result` outputs before acting (`validation-zod.md`) and verify high-stakes subagent claims against sources (`subagents.md`).
- Minimize context handed to subagents to limit what an injection can exfiltrate.

## Tenant isolation

Put `tenant_id` on every multi-tenant table, filter every query by it, and bind it into tools/agents from the authenticated principal — never from model input (`database.md`). Choose agent instance `id` semantics that don't let one tenant address another's instance.

## Channel ingress

Channel signature/issuer/audience/tenant verification is the trust boundary (`channels.md`) — don't weaken it. Because providers retry and Flue doesn't dedupe, claim activity ids in durable storage before dispatch to prevent duplicate side effects.

## Data retention

Flue persists session messages and run events (DO SQLite / your adapter). Decide retention deliberately, keep business/PII data in your own store with its own retention controls, and review external observability backends' retention before exporting model content (which may include user data).

## Anti-patterns

- Implicit `/` mount with no auth middleware in production.
- 401/403 on run reads (discloses existence).
- Secrets/tenant ids as model parameters.
- Trusting model output or sandbox files as authoritative without validation.
- Unbounded outbound tools an injection can redirect.
