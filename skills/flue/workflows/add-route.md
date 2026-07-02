# Recipe: add a custom route / compose the HTTP surface

**Purpose.** Add your own Hono routes and middleware around Flue's mounted surface, or mount Flue under a prefix.

**Inputs.** The custom paths you need; auth policy; whether Flue mounts at `/` or `/api`.
**Outputs.** `src/app.ts` exporting a Hono app that composes `flue()`.
**Success criteria.** Custom routes respond; Flue routes work; middleware rejects unauthorized requests; run reads return 404 (not 401/403) for unauthorized callers.

## Steps

1. **Confirm the API:** `pnpm flue docs` ("routing", "app.ts"). See `../reference/routing.md`.
2. **Create/extend** `src/app.ts`:
   ```ts
   import { flue } from '@flue/runtime/routing';
   import { Hono, type MiddlewareHandler } from 'hono';
   import { authenticate } from './lib/auth.ts';

   const requireUser: MiddlewareHandler = async (c, next) => {
     const user = await authenticate(c.req.raw);
     if (!user) { return c.json({ error: 'Unauthorized' }, 401); }
     await next();
   };

   const app = new Hono();
   app.get('/health', (c) => c.json({ ok: true }));
   app.use('/agents/*', requireUser);
   app.use('/workflows/*', requireUser);
   app.use('/runs/*', requireUser);
   app.use('/channels/*', requireUser);
   app.route('/', flue());        // or app.route('/api', flue())
   export default app;
   ```
3. **If mounting under `/api`,** update SDK clients to `createFlueClient({ baseUrl: '.../api' })` and add `/api/*` + `/_flue/*` to `run_worker_first` when serving assets (`../reference/hono-workers.md`).
4. **Custom webhooks** use `dispatch()` (`../reference/channels.md`).

## Verification

```bash
pnpm flue build --target <node|cloudflare>
pnpm flue dev
curl http://localhost:3583/health                 # -> {"ok":true}
curl -i http://localhost:3583/workflows/<name>     # -> 401 without auth
curl -i http://localhost:3583/runs/does-not-exist  # -> 404, not 401/403
```

## References
`../reference/routing.md`, `../reference/hono-workers.md`, `../reference/security.md`
