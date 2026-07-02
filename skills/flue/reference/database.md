# Database and persistence

## Contents

- [The persistence boundary (read first)](#the-persistence-boundary-read-first)
- [Flue's own store](#flues-own-store)
- [Your application's business data](#your-applications-business-data)
- [Access patterns](#access-patterns)
- [Idempotency storage](#idempotency-storage)
- [Anti-patterns](#anti-patterns)

## The persistence boundary (read first)

Flue distinguishes **its own state** from **your business data**, and they live in different places.

> Flue persists agent session snapshots and messages, accepted prompts and `dispatch()` submissions, workflow-run records and events, and run indexing. It does **not** store sandbox files, external side effects, application business logic, or credentials — and a persisted session does **not** make a sandbox durable. Keep customer records, payments, tickets, and other business data in **your own application database**.

## Flue's own store

You don't manage Flue's session/run store directly — it's configured per target:

- **Cloudflare Workers** — uses built-in **Durable Object SQLite automatically**. There is no `db.ts`; it's wired in for you.
- **Node.js** — provide `src/db.ts` exporting a `PersistenceAdapter`:
  ```ts
  // src/db.ts (Node) — SQLite
  import { sqlite } from '@flue/runtime/node';
  export default sqlite('./data/flue.db');
  ```
  ```ts
  // src/db.ts (Node) — Postgres
  import { postgres } from '@flue/postgres';
  export default postgres(process.env.DATABASE_URL!);
  ```
  Flue discovers `db.ts` at build time and wires the exported adapter into the generated server entry.

`flue add` can fetch database installation blueprints — run `pnpm flue docs` for the adapters available in your version.

## Your application's business data

Durable business data (customers, tickets, orders, billing, audit logs) belongs in a store **you** own, accessed the normal way for your target — a Postgres/MySQL/etc. connection on Node, or a standard Workers binding on Cloudflare. This is separate from Flue's persistence above; do not push business records into agent session state.

Keep queries and schema out of the discovered `agents/`/`workflows/` folders, e.g.:

```
src/
├─ db/
│  ├─ tickets.ts   # query functions (always parameterized, tenant-scoped)
│  └─ schema.sql   # reference snapshot of current schema
└─ tools/ticket-tools.ts  # tools that wrap db/ functions with tenant scope
```

## Access patterns

Reach your business store through `c.env`/binding in Hono routes (`routing.md`) or `ctx.env` in workflow `run`, and expose it to agents only through a **narrow, tenant-scoped tool** (`tools.md`) — never hand the model raw database access:

```ts
import { defineTool } from '@flue/runtime';
import { z } from 'zod';

export function createTicketTools(db: TicketStore, tenantId: string) {
  const getTicket = defineTool({
    name: 'get_ticket',
    description: 'Fetch one ticket by id for the current tenant.',
    parameters: z.object({ ticketId: z.string() }),
    execute: async ({ ticketId }) => {
      const row = await db.getById(ticketId, tenantId); // bind tenantId in code, not via the model
      return row ? JSON.stringify(row) : 'No ticket found.';
    },
  });
  return [getTicket];
}
```

Always parameterize queries and scope every read/write by the authenticated tenant (`security.md`).

## Idempotency storage

Flue does not document native workflow retries or channel deduplication, and event providers retry on non-2xx (`channels.md`). When duplicate admission is unacceptable, claim the event id in **your own durable store** before dispatch and skip if already claimed:

```ts
const firstSeen = await db.claimOnce(event.id); // INSERT ... ON CONFLICT DO NOTHING, returns false if seen
if (!firstSeen) { return c.json({ ok: true, duplicate: true }, 202); }
// ... dispatch the event ...
```

## Anti-patterns

- Treating any external DB as Flue's session/run store (that's DO SQLite on Cloudflare, or your `db.ts` adapter on Node).
- Storing business records in agent session state instead of your own database.
- Giving the model raw DB access instead of a narrow, tenant-scoped tool.
- String-interpolating SQL, or omitting tenant scoping in multi-tenant tables.
