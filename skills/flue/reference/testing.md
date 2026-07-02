# Testing and evals

## Contents

- [Strategy](#strategy)
- [Unit and integration tests](#unit-and-integration-tests)
- [Evals with vitest-evals](#evals-with-vitest-evals)
- [Running evals](#running-evals)
- [Testing a deployed app](#testing-a-deployed-app)
- [Keeping examples runnable](#keeping-examples-runnable)
- [Pre-deploy checklist](#pre-deploy-checklist)
- [Anti-patterns](#anti-patterns)

## Strategy

Test in layers: pure functions and tool `execute` bodies with ordinary unit tests; HTTP routes and middleware as integration tests against `flue dev`; and agent/workflow *behavior* with **evals** (repeatable scenarios + expectations) since model output is non-deterministic. Evals decide whether new behavior is acceptable before shipping.

## Unit and integration tests

- **Tools** — call `tool.execute(args)` directly with a fake binding/store and assert the returned string. Validate that bad input is rejected by the schema.
- **Your routes** — start `flue dev` (or use Vitest with the Workers pool) and exercise `POST /workflows/:name?wait=result` and the agent endpoints; assert status codes and bodies, including auth (401 for anonymous, 404 for unauthorized run reads — see `security.md`).
- **Schemas** — unit-test request/result schemas with representative good and bad payloads.

## Evals with vitest-evals

Flue recommends Sentry's **vitest-evals**. Scaffold the harness:

```bash
flue add tooling vitest-evals
```

This generates `createFlueAgentHarness(...)`, which runs agents through the SDK and captures responses, token usage, costs, and tool interactions in vitest-evals format.

```ts
import { expect } from 'vitest';
import { describeEval, toolCalls } from 'vitest-evals';
import { createFlueAgentHarness } from './harness.ts';

const harness = createFlueAgentHarness({ agentName: 'service-status' });

describeEval('Flue service status agent', { harness }, (it) => {
  it('checks live service status before answering', async ({ run }) => {
    const result = await run('Is the checkout service currently operational?');

    expect(result.output).toContain('operational');
    expect(toolCalls(result).map((call) => call.name)).toContain('get_service_status');
    expect(result.usage.totalTokens).toBeGreaterThan(0);
  });
});
```

Assert on direct output (`result.output`), tool usage (`toolCalls(result)`), and usage metrics (`result.usage.totalTokens`). For subjective qualities (clarity, faithfulness), use `toSatisfyJudge(...)` with a judge config.

## Running evals

```bash
pnpm run evals          # development run
pnpm run evals:json     # JSON report
```

The eval suite expects the Flue dev server running in another terminal:

```bash
pnpm exec flue dev
```

For non-vitest setups, call agents/workflows directly via `client.agents.prompt(...)` / `client.workflows.invoke(...)` and score with any framework.

## Testing a deployed app

```bash
FLUE_BASE_URL=https://your-app.example.com pnpm run evals
```

## Keeping examples runnable

Every code example you add to a Flue codebase (including in skills/docs) should be import-checked by `pnpm flue build` and, where it's a workflow, exercised by a smoke `curl`. Prefer copying examples into a real file and building over trusting them by eye.

## Pre-deploy checklist

1. `pnpm flue build --target <node|cloudflare>` — clean build.
2. Unit tests pass.
3. `pnpm run evals` green against `flue dev`.
4. (Cloudflare) `wrangler deploy --dry-run --config dist/<name>/wrangler.json` passes.
5. New agents/workflows have a migration tag (`deployment-cloudflare.md`).
6. Smoke-test the affected route with `curl`.

## Anti-patterns

- Asserting exact model strings instead of substrings/judges (flaky).
- Running evals without the dev server (they need it).
- Shipping without a build + smoke test.
- Treating a single passing eval as proof; cover failure scenarios too.
