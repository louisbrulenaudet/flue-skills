# Channels

## Contents

- [What a channel is](#what-a-channel-is)
- [Adding a channel](#adding-a-channel)
- [Signature verification](#signature-verification)
- [Normalization and envelopes](#normalization-and-envelopes)
- [Routing into agents via dispatch](#routing-into-agents-via-dispatch)
- [Idempotency and retries](#idempotency-and-retries)
- [Outbound message tool](#outbound-message-tool)
- [Configuration](#configuration)
- [Anti-patterns](#anti-patterns)

## What a channel is

A channel is a **verified provider ingress** module: it receives authenticated activities from an external service (Teams, GitHub, …), verifies them, normalizes them into a dispatch envelope, and routes them into agent instances. Channels publish their HTTP surface automatically under `/channels/:name/<suffix>` — no `route` export needed.

This page uses the **Teams** channel as the concrete example (the documented reference); other providers follow the same shape.

## Adding a channel

```bash
flue add channel teams
```

This installs `@flue/teams`, creates `lib/teams-client.ts` (a fetch-based OAuth client), and generates `channels/teams.ts` (the channel module plus an outbound message tool). Review the generated files — you own them after scaffolding.

## Signature verification

The channel authenticates before processing. Teams, for example, validates the Microsoft OpenID signing key and `RS256` signature; checks issuer, application audience, and expiration; verifies the signing key's `msteams` endorsement; matches the activity's exact `serviceUrl` against the signed token claim; and enforces the conversation/channel tenant against `TEAMS_TENANT_ID`. Treat the channel's verification as the trust boundary — do not weaken it.

## Normalization and envelopes

The handler switches on the provider-native event type (`message`, `conversationUpdate`, `invoke`, …) and extracts fields using documented provider property names, producing a normalized envelope:

```ts
{
  type: 'teams.message',
  activityId: activity.id,
  sender: activity.from,
  text: activity.text,
  entities: activity.entities,
}
```

Keep the envelope shape stable — agents and workflows downstream depend on it.

## Routing into agents via dispatch

The channel hands normalized events to an agent with `dispatch()`, deriving the instance id from the activity's destination so the same conversation maps to the same agent instance:

```ts
await dispatch(assistant, {
  id: channel.conversationKey(channel.destination(activity)),
  input: { /* normalized envelope */ },
});
```

## Idempotency and retries

The provider connector **retries on any non-2xx response**, and the channel package **does not deduplicate activity ids**. So:

- **Admit fast.** `dispatch(...)` the activity and return `202` quickly; do not block the response on the agent finishing.
- **Deduplicate yourself** when duplicate admission is unacceptable: claim the activity id in application-owned durable storage (your own database, see `database.md`) *before* dispatch, and skip if already claimed.

## Outbound message tool

The generated tool lets an agent send replies. Bind routing credentials/authorization in trusted code; the model only chooses the message text:

```ts
import { createAgent } from '@flue/runtime';
import { channel, postMessage } from '../channels/teams.ts';

export default createAgent(({ id }) => ({
  model: 'anthropic/claude-haiku-4-5',
  tools: [postMessage(channel.parseConversationKey(id))],
}));
```

## Configuration

Teams uses these env vars (set via secrets — see `deployment-cloudflare.md`):

| Variable           | Purpose                                  |
| ------------------ | ---------------------------------------- |
| `TEAMS_APP_ID`     | Constrains inbound JWT audience.         |
| `TEAMS_TENANT_ID`  | Enforces activity tenant identity.       |
| `TEAMS_APP_PASSWORD` | Authenticates outbound OAuth requests. |

Point the provider's webhook endpoint at `https://<host>/channels/teams/activities`.

## Anti-patterns

- Blocking the webhook response on agent completion (causes provider retries → duplicates).
- Relying on the framework to dedupe activity ids — it does not.
- Letting the model supply routing/credentials instead of binding them in code.
- Loosening signature/tenant checks to "make it work" in testing.
