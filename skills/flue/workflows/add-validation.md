# Recipe: add runtime validation

**Purpose.** Validate inputs/outputs at trust boundaries — tool inputs, structured agent/workflow results, and your own HTTP bodies.

> This project uses **Zod** everywhere. Flue's own docs show valibot for tool `parameters`/`result`; Zod fits via Standard Schema, with `z.toJSONSchema()` as a fallback for `parameters`. See `../reference/validation-zod.md`.

**Inputs.** The shape to validate and where it crosses a boundary.
**Outputs.** A co-located schema + inferred type; parsing wired in with structured error handling.
**Success criteria.** Bad input is rejected with a clear error; good input flows through typed; no hand-maintained parallel type.

## Steps

1. **Identify the boundary:** tool input, `result` output, or your HTTP route body.
2. **Tool input (Zod):**
   ```ts
   parameters: z.object({ orderId: z.string().describe('order_1234') }),
   // fallback if your version needs JSON Schema: z.toJSONSchema(z.object({ orderId: z.string() }))
   ```
3. **Structured result (Zod):**
   ```ts
   const r = await session.prompt(text, { result: z.object({ priority: z.enum(['low','medium','high']), summary: z.string() }) });
   return r.data;
   ```
4. **Your HTTP body (Zod):**
   ```ts
   const Body = z.object({ text: z.string().min(1) });
   export type Body = z.infer<typeof Body>;
   const parsed = Body.safeParse(await c.req.json());
   if (!parsed.success) { return c.json({ error: parsed.error.issues }, 400); }
   ```
5. **Infer the type** from the schema (`z.infer`); never write a parallel `type`. Co-locate the schema with the code that owns the shape.

## Verification

```bash
pnpm flue build --target <node|cloudflare>
pnpm flue dev
# good payload -> success; bad payload -> 400 with issues (your boundary) or schema rejection (tool/result)
curl -i "http://localhost:3583/workflows/<wf>?wait=result" -d '{"bad":"shape"}'
```

## References
`../reference/validation-zod.md`, `../reference/tools.md`, `../reference/workflows.md`
