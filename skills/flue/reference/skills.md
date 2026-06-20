# Flue skills

## Contents

- [What a Flue skill is](#what-a-flue-skill-is)
- [File structure](#file-structure)
- [Frontmatter](#frontmatter)
- [Importing skills into agents](#importing-skills-into-agents)
- [Workspace auto-discovery](#workspace-auto-discovery)
- [Invoking a skill from a workflow](#invoking-a-skill-from-a-workflow)
- [Skill vs reference file vs tool](#skill-vs-reference-file-vs-tool)
- [Authoring guidance](#authoring-guidance)
- [Anti-patterns](#anti-patterns)

## What a Flue skill is

A Flue skill is **reusable instructions and supporting resources** an agent loads for specialized, repeatable work. Skills shape behavior; they do **not** execute code — for actions, use tools or the sandbox (`tools.md`). (This is the same model as the Claude/agent-skills format you are reading now.)

## File structure

Each skill is a folder with a `SKILL.md` and optional `references/`:

```
src/skills/
└─ review/
   ├─ SKILL.md
   └─ references/
      └─ checklist.md
```

## Frontmatter

`SKILL.md` requires YAML frontmatter. Validated fields:

- **`name`** — lowercase letters, numbers, hyphens; no leading/trailing/consecutive hyphens; ≤ 64 chars.
- **`description`** — non-empty, ≤ 1024 chars. This is how the agent decides relevance — write it in the third person and make it specific.
- Optional informational: `license`, `compatibility`, `metadata`.

```markdown
---
name: review
description: >-
  Reviews a proposed change for correctness, security, and style, returning an
  approve/reject decision with a short rationale. Use before merging.
---

# Review

1. Read the change and any linked context.
2. Check correctness, then security, then style.
3. Return a decision and a one-paragraph rationale.

See references/checklist.md for the full gate list.
```

## Importing skills into agents

Import with the `skill` attribute and pass to `skills`:

```ts
import { createAgent } from '@flue/runtime';
import review from '../skills/review/SKILL.md' with { type: 'skill' };
import triage from '../skills/triage/SKILL.md' with { type: 'skill' };

export default createAgent(() => ({
  model: 'anthropic/claude-sonnet-4-6',
  skills: [review, triage],
}));
```

Skills can also come from installed packages:

```ts
import review from '@acme/review-skills/review/SKILL.md' with { type: 'skill' };
```

## Workspace auto-discovery

Flue automatically discovers workspace skills (no import) from `<cwd>/.agents/skills/` directories, making them available by their declared `name`. Use this when an agent's sandbox workspace ships its own skills.

## Invoking a skill from a workflow

Trigger a skill explicitly through the session API, optionally validating its output:

```ts
import { z } from 'zod';

const response = await session.skill('review', {
  args: { change: payload.change },
  result: z.object({ approved: z.boolean(), summary: z.string() }),
});
// omit `result` for text-only responses
```

## Skill vs reference file vs tool

- **Skill** — a repeatable *process/instructions* the agent invokes by name across many runs.
- **Reference file** (inside a skill's `references/`) — supporting detail loaded on demand; not independently invokable.
- **Tool** — executable code (`tools.md`). If you need an *action*, it's a tool, not a skill.

## Authoring guidance

Follow progressive disclosure: keep `SKILL.md` concise and high-signal, push depth into `references/`, and give each skill one clear workflow. A precise `description` is the single biggest factor in whether the agent picks the right skill.

## Anti-patterns

- One oversized `SKILL.md` instead of a tight entrypoint plus references.
- Vague descriptions ("helps with reviews") that don't aid selection.
- Putting executable logic in a skill — that belongs in a tool.
- Deeply nested reference trees; keep references one level under the skill.
