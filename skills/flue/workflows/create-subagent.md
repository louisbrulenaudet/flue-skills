# Recipe: create a subagent

**Purpose.** Give a parent agent a bounded specialist (research, review, DB lookups) it can delegate to. Subagents have no public endpoint.

**Inputs.** Subagent role + name; instructions; the narrow tools it needs; expected output shape.
**Outputs.** A `defineAgentProfile(...)` added to the parent's `subagents` array.
**Success criteria.** The parent delegates to it; the subagent returns the expected structured result using only its allowed tools.

## Steps

1. **Confirm the API:** `pnpm flue docs` ("subagents", "defineAgentProfile"). See `../reference/subagents.md`.
2. **Define the profile** with `name` + `description` (so the parent can select it) and least-privilege tools:
   ```ts
   import { defineAgentProfile } from '@flue/runtime';

   const reviewer = defineAgentProfile({
     name: 'change_reviewer',
     description: 'Reviews a diff for correctness and security; returns approve/reject with reasons.',
     instructions: 'Apply the checklist. Return { approved, issues[] } only — no prose.',
     // tools: [readOnlyDiffTool],
   });
   ```
3. **Attach** to the parent: `subagents: [reviewer]`.
4. **Shape the output** and validate it (`result` schema) where the parent acts on it.

## Verification

```bash
pnpm flue build --target <node|cloudflare>
pnpm flue dev
# Prompt the parent with a task that should trigger delegation:
pnpm flue connect <parent-agent> demo
# Confirm via run inspection that the subagent ran and returned the expected shape:
pnpm flue logs <runId> --server http://localhost:3583
```
Check: the subagent used only its allowed tools; the parent verified high-stakes claims before acting.

## References
`../reference/subagents.md`, `../reference/agents.md`, `../reference/security.md`
