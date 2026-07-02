# Recipe: create a Flue skill

**Purpose.** Add reusable instructions an agent loads for repeatable work (a skill shapes behavior; it does not execute code).

**Inputs.** Skill name (kebab-case); a precise description; the process steps; optional reference files.
**Outputs.** `src/skills/<name>/SKILL.md` (+ optional `references/`), imported into an agent's `skills` array.
**Success criteria.** Skill builds and imports; the agent loads it by name and follows it; output validates when invoked from a workflow.

## Steps

1. **Confirm the API:** `pnpm flue docs` ("skills", "type: 'skill'"). See `../reference/skills.md`.
2. **Create the folder and `SKILL.md`:**
   ```markdown
   ---
   name: review
   description: >-
     Reviews a change for correctness, security, and style and returns an
     approve/reject decision with a short rationale. Use before merging.
   ---

   # Review

   1. Read the change and any linked context.
   2. Check correctness, then security, then style.
   3. Return a decision and a one-paragraph rationale. See references/checklist.md.
   ```
   `name`: kebab-case, ≤64 chars. `description`: ≤1024 chars, third person, specific.
3. **Push depth into `references/`** (progressive disclosure); keep `SKILL.md` tight.
4. **Import into an agent:**
   ```ts
   import review from '../skills/review/SKILL.md' with { type: 'skill' };
   export default createAgent(() => ({ model: 'anthropic/claude-sonnet-4-6', skills: [review] }));
   ```
5. **(Optional) invoke from a workflow** with validation:
   ```ts
   const r = await session.skill('review', { args: { change }, result: z.object({ approved: z.boolean(), summary: z.string() }) });
   ```

## Verification

```bash
pnpm flue build --target <node|cloudflare>   # import + frontmatter validated
pnpm flue dev
pnpm flue connect <agent> demo               # confirm the agent applies the skill
```

## References
`../reference/skills.md`, `../reference/agents.md`, `../reference/workflows.md`
