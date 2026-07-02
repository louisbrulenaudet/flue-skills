# Flue skill pack — index

Operational knowledge for building Flue AI-agent apps on Cloudflare Workers with Hono and the Agents SDK. Start at [`SKILL.md`](./SKILL.md); it routes to the right file by task.

## Layout

- **`SKILL.md`** — concise entrypoint, decision tree, philosophy, global rules.
- **`reference/`** — one focused file per Flue domain (depth, conventions, anti-patterns).
- **`workflows/`** — step-by-step recipes with inputs, outputs, success criteria, and a verification step.
- **`checks/eval-cases.md`** — representative tasks to validate the pack against before relying on it.

## How an agent should use this pack

1. Read `SKILL.md` and pick the sub-skill from the table / decision tree.
2. For a concrete job, open the matching `workflows/*.md` recipe and follow it end to end.
3. Pull supporting depth from `reference/*.md` only as needed (progressive disclosure — don't load everything).
4. **Confirm any API against the installed Flue version** via `pnpm flue docs` or https://flueframework.com/docs before writing code.
5. Run the recipe's verification step before reporting done.

## Source of truth

All content is grounded in the official Flue documentation (https://flueframework.com/docs) as read during authoring. This project standardizes on **Zod** for validation (Flue's own examples use valibot; `validation-zod.md` explains how Zod fits Flue's schema slots). Where the docs are silent (e.g. native workflow retries/idempotency), this pack says so explicitly and frames guidance as a recommendation rather than fact.
