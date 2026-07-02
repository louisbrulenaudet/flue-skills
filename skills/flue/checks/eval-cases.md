# Evaluation cases for this skill pack

Run these representative tasks against the pack before relying on it. Each case lists the prompt, the file the agent should reach for, and pass/fail criteria targeting a known failure mode. A case passes only when the agent produces correct Flue code **and** runs the recipe's verification step.

## How to use

For each case: give the prompt to an agent that has this skill loaded, observe which reference/workflow file it opens, and score against the criteria. Track regressions when the pack or the Flue version changes.

| # | Prompt | Should route to | Pass criteria | Failure mode guarded |
|---|--------|-----------------|---------------|----------------------|
| 1 | "Scaffold a new Flue project for Cloudflare." | `workflows/build-deploy-cloudflare.md`, `reference/deployment-cloudflare.md` | Installs `@flue/runtime zod 'agents@^0.14.1'` + dev deps; `flue init --target cloudflare`; correct `wrangler.jsonc`. | Wrong deps / missing Agents SDK. |
| 2 | "Add an agent with a tool that looks up an order." | `workflows/create-agent.md`, `reference/tools.md` | `createAgent` default export + `route`; `defineTool` with a Zod `parameters` schema (direct or `z.toJSONSchema`); tool attached. | Inventing an undocumented tool API. |
| 3 | "Expose a summarize workflow over HTTP returning {summary}." | `workflows/create-workflow.md` | `run(ctx)` + `WorkflowRouteHandler`; `?wait=result`; doesn't call `run()` directly. | Calling `run()` directly; missing `route`. |
| 4 | "Where does Flue store sessions on Cloudflare?" | `reference/database.md` | States DO SQLite is Flue's automatic store; business data goes in your own DB. | Wiring an external DB as Flue's session store. |
| 5 | "Persist tickets and let the agent query them." | `reference/database.md`, `reference/tools.md` | Business data in your own DB; access via a parameterized, tenant-scoped tool. | SQL injection / no tenant scope / business data in session. |
| 6 | "Deploy my Flue worker." | `workflows/build-deploy-cloudflare.md` | Deploys `dist/<name>/wrangler.json`; appends migration tag with `new_sqlite_classes`. | Deploying source `wrangler.jsonc`; missing/edited migration. |
| 7 | "Mount Flue under /api behind auth." | `workflows/add-route.md`, `reference/routing.md` | `src/app.ts` composes `flue()`; middleware on `/agents/* …`; SDK baseUrl updated; assets `run_worker_first` includes `/_flue/*`. | Implicit `/` mount with no auth; broken assets routing. |
| 8 | "Accept Teams messages into an agent." | `workflows/add-channel.md`, `reference/channels.md` | `flue add channel teams`; dedup before dispatch; 202 fast; secrets set. | No idempotency; blocking the webhook response. |
| 9 | "Validate the request body for /translate." | `workflows/add-validation.md`, `reference/validation-zod.md` | Zod at the HTTP boundary with `safeParse`+400; Zod for Flue `parameters`/`result` (or `z.toJSONSchema`); inferred types. | Hand-maintained types; `.parse()` throwing at the boundary. |
| 10 | "Add a research specialist the assistant delegates to." | `workflows/create-subagent.md`, `reference/subagents.md` | `defineAgentProfile` with name/description; least-privilege tools; output verified. | Public endpoint for the subagent; broad tools. |
| 11 | "Make this agent reusable across two endpoints." | `reference/agents.md` (profiles) | `defineAgentProfile` + `profile`. | Duplicating config. |
| 12 | "How do I know a workflow failed in prod?" | `reference/observability.md` | `observe()` `run_end`/`isError`; `flue logs <runId>`; structured `ctx.log`. | No failure monitoring. |
| 13 | "Write tests for my service-status agent." | `reference/testing.md` | `flue add tooling vitest-evals`; `describeEval`/`toolCalls`; dev server running. | Asserting exact model text; no dev server. |
| 14 | "Add a skill that encodes our review checklist." | `workflows/create-skill.md`, `reference/skills.md` | `SKILL.md` with valid frontmatter; `with { type: 'skill' }`; progressive disclosure. | Executable logic in a skill; oversized SKILL.md. |
| 15 | "Pick a model for cheap high-volume triage." | `reference/agents.md` (model selection) | `provider/model` string; smaller model; `thinkingLevel` if needed. | Invented model id / wrong format. |

## Cross-cutting checks (apply to every case)

- The agent confirmed the API against `pnpm flue docs` / the live docs rather than memory.
- It did not invent undocumented Flue APIs, bindings, or file paths.
- It used Zod for validation (direct schema, or `z.toJSONSchema()` for Flue `parameters` when needed).
- It did not wire an external database in as Flue's session store.
- It ran the recipe's verification step before reporting done.
- It kept `SKILL.md` / `CLAUDE.md` lean and used progressive disclosure.
