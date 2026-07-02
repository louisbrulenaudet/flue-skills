# Observability

## Contents

- [Three mechanisms](#three-mechanisms)
- [Structured logging with ctx.log](#structured-logging-with-ctxlog)
- [Application-wide observe()](#application-wide-observe)
- [Run inspection](#run-inspection)
- [Providers: OpenTelemetry, Braintrust, Sentry](#providers-opentelemetry-braintrust-sentry)
- [Request correlation](#request-correlation)
- [Data export guidance](#data-export-guidance)
- [Anti-patterns](#anti-patterns)

## Three mechanisms

Flue offers run inspection (per-`runId` history), the `observe()` API (application-wide activity), and structured logging (`ctx.log`). Use logs for application-specific signal, `observe()` for cross-cutting monitoring, and run inspection for debugging individual runs.

## Structured logging with ctx.log

Inside a workflow `run`, log with structured attributes via `ctx.log`:

```ts
import { createAgent, type FlueContext } from '@flue/runtime';

const summarizer = createAgent(() => ({
  model: 'anthropic/claude-haiku-4-5',
  instructions: 'Summarize the supplied document clearly and concisely.',
}));

export async function run({ init, log, payload }: FlueContext<{ text: string }>) {
  log.info('Summarization requested', { characters: payload.text.length });

  const harness = await init(summarizer);
  const session = await harness.session();
  const response = await session.prompt(payload.text);

  log.info('Summarization completed', {
    tokens: response.usage.totalTokens,
    cost: response.usage.cost.total,
  });
  return { summary: response.text };
}
```

`log.info`, `log.warn`, `log.error` all take a message plus a structured-attributes object. Log events are persisted into the run stream during a run, so they show up in run inspection.

## Application-wide observe()

Register `observe()` in your entrypoint (`src/app.ts`) to monitor all workflows and continuing agents. Treat every event as read-only:

```ts
import { observe } from '@flue/runtime';
import { flue } from '@flue/runtime/routing';
import { Hono } from 'hono';

observe((event) => {
  if (event.type === 'run_end' && event.isError) {
    console.error('Workflow failed', event.runId, event.error);
  }
  if (event.type === 'operation' && event.durationMs > 5_000) {
    console.warn('Slow operation', event.operationKind, event.durationMs);
  }
  if (event.type === 'log' && event.level === 'error') {
    console.error(event.message, event.attributes);
  }
});

const app = new Hono();
app.route('/', flue());
export default app;
```

Event types include `run_start`, `run_end` (with `isError`), `operation` (`operationKind`, `durationMs`), `log` (`level`, `message`, `attributes`), `message_end`, and streaming deltas.

## Run inspection

```bash
pnpm exec flue logs <runId> --server http://localhost:3583
```

Programmatically (`workflows.md`): `GET /runs/<runId>` streams events; `?meta` returns metadata; the SDK exposes `client.runs.get()/.events()/.stream()`.

## Providers: OpenTelemetry, Braintrust, Sentry

Flue supports external telemetry through adapters/observers:

| Provider       | Best for                                                                 |
| -------------- | ------------------------------------------------------------------------ |
| OpenTelemetry  | Vendor-neutral traces, or an existing OTel-compatible backend (`@flue/opentelemetry`). |
| Braintrust     | Content-bearing traces, model usage, costs, eval debugging.              |
| Sentry         | Actionable failures and explicit error logs without exporting model content by default. |

Wire these via the documented adapter/`observe(...)` integration for your version — confirm exact setup with `pnpm flue docs`.

## Request correlation

Correlate by `runId` (every run has one) and attach your own ids (request id, tenant id, ticket id) as structured `log` attributes so traces and logs join up across the HTTP boundary, the run, and downstream tools.

## Data export guidance

- Flue replaces image data with omission sentinels before events are observed — but **you** must sanitize payloads, log attributes, tool details, and results before exporting.
- When aggregating usage, sum the model-turn **leaf** values, not operation roll-ups, to avoid double-counting.
- Review each backend's retention, access, and redaction controls before exporting model content.

## Anti-patterns

- Logging secrets, full prompts, or PII into attributes that get exported.
- Mutating events inside `observe()` (treat them as read-only).
- Summing operation roll-ups for usage/cost (double counts).
- Shipping with no `run_end`/`isError` handler, so failures go unnoticed.
