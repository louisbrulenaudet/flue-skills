# Subagents

## Contents

- [What a subagent is](#what-a-subagent-is)
- [Defining a subagent](#defining-a-subagent)
- [Delegation boundaries](#delegation-boundaries)
- [Specialist examples](#specialist-examples)
- [Prompt hygiene](#prompt-hygiene)
- [Verifying subagent output](#verifying-subagent-output)
- [Subagent vs tool vs workflow](#subagent-vs-tool-vs-workflow)
- [Anti-patterns](#anti-patterns)

## What a subagent is

A subagent is an **agent profile the parent delegates to**. It has no public HTTP endpoint and is callable only by its parent. Use it to give a parent a bounded specialist (research, review, database lookups) with its own instructions and tools, keeping the parent's context clean.

## Defining a subagent

Create profiles with `defineAgentProfile(...)` (include `name` + `description` so the parent can select them) and pass them in `subagents`:

```ts
import { createAgent, defineAgentProfile } from '@flue/runtime';

const policyResearcher = defineAgentProfile({
  name: 'policy_researcher',
  description: 'Finds relevant policy text and quotes supporting passages.',
  instructions: 'Read the policy workspace and return supporting quotations with file paths.',
});

export default createAgent(() => ({
  model: 'anthropic/claude-sonnet-4-6',
  subagents: [policyResearcher],
}));
```

A subagent profile takes the same shape as any agent profile (`model`, `instructions`, `tools`, …); the `name`/`description` make it addressable for delegation.

## Delegation boundaries

Give each subagent **one job and only the tools that job needs**. The parent owns orchestration and the final answer; subagents return scoped results, not user-facing prose. Pass a subagent only the context it needs — not the whole conversation — to keep it focused and to limit data exposure (`security.md`).

## Specialist examples

- **Research subagent** — `description: 'Searches the knowledge base and returns sourced findings.'`; tools: read-only search. Returns quotes + source ids.
- **Review subagent** — `description: 'Reviews a diff for correctness and security and returns a verdict.'`; instructions: a fixed checklist; returns `{ approved, issues[] }`.
- **Database subagent** — `description: 'Answers questions from the orders database via safe, parameterized reads.'`; tools: a narrow, read-only query tool bound to the tenant (`tools.md`). Never give it write access unless the task requires it.

## Prompt hygiene

Keep subagent instructions short, imperative, and output-shaped ("Return X as Y"). Specify the expected return structure and validate it with a `result` schema where possible. Avoid leaking the parent's full system prompt into the subagent.

## Verifying subagent output

A subagent's output is untrusted until checked. Where it drives a decision:

- Validate the shape with a Zod `result` schema (see `validation-zod.md`).
- For high-stakes results, have the parent (or a second review subagent) sanity-check the claim against the source before acting.
- Surface source ids/quotes so the parent can confirm rather than trust.

## Subagent vs tool vs workflow

| Need                                              | Use            |
| ------------------------------------------------- | -------------- |
| A single deterministic action                     | a **tool**     |
| A multi-step specialist with reasoning, parent-driven | a **subagent** |
| A finite, independently-admitted job over HTTP    | a **workflow** |

## Anti-patterns

- Giving a subagent broad tools/credentials "just in case."
- Letting a subagent write the final user-facing answer instead of returning structured findings.
- Passing the entire conversation to every subagent.
- Acting on subagent output without validating its shape or its claims.
