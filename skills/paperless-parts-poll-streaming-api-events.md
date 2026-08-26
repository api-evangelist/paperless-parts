---
name: paperless-parts-poll-streaming-api-events
description: >-
  Consume the Paperless Parts Streaming API — a poll-based event feed scoped to a managed
  integration — and turn each event into an integration action, without double-processing.
api: Paperless Parts API v1
version: '1.0'
generated: '2026-08-26'
method: generated
source: openapi/paperless-parts-v1-openapi.yml
operations:
  - ListManagedIntegrations
  - CreateManagedIntegration
  - PostManagedIntegrationHeartbeat
  - PollEvents
  - ListEvents
  - CreateIntegrationAction
  - ListIntegrationActions
---

# React to Paperless Parts events

Paperless Parts calls this the **Streaming API**, but nothing is pushed. It is a poll feed, and it
lives **only in v1** — `https://api.paperlessparts.com/v1`. There are no webhooks, no callback
registration, and no signature header.

## Steps

1. **Register (or find) your managed integration.** `GET /managed_integrations/public`
   (`ListManagedIntegrations`) or `POST /managed_integrations/public`
   (`CreateManagedIntegration`). Keep the returned `managed_integration_uuid` — every event call is
   scoped to it.
2. **Poll.** `GET /managed_integrations/public/{managed_integration_uuid}/poll` (`PollEvents`)
   returns events that have not yet been dispatched for this integration.
   - Leave `create_dispatch_records` unset or set it to `true` and the returned events will **not**
     appear in the next response — the dispatch record is the cursor.
   - Set it to `false` to peek without consuming. Use this while developing so you do not burn
     events you have not handled yet.
   - Narrow with `event_type_in`.
3. **Handle each event.** The `Event` object is `{uuid, type, data, related_object,
   related_object_type, created}`. `related_object` is the UUID of the thing the event is about and
   `related_object_type` tells you what it is; `type` is a dotted string such as
   `integration_action.requested`.
4. **Record an integration action.** `POST /managed_integrations/public/{uuid}/integration_actions`
   (`CreateIntegrationAction`) — the docs are explicit that the event id is what you use to create
   an integration action. Read them back with `ListIntegrationActions`, and update status with
   `PATCH /integration_actions/public/{integration_action_uuid}`.
5. **Heartbeat.** `POST /managed_integrations/public/{uuid}/heartbeat`
   (`PostManagedIntegrationHeartbeat`) so Paperless Parts can show the integration as alive in
   Integration Manager.

## Rules that matter here

- **v1 only.** v2 does not publish the Events or Integration Actions endpoints. If you also need
  jobs, parts or granular quote composition, you will be calling both versions.
- **The event-type vocabulary is not published.** The OpenAPI carries exactly one example value
  (`integration_action.requested`) and `event_type_in` is an unconstrained string. Do not hard-fail
  on an unrecognised `type` — log it and continue.
- **No poll interval is documented.** No recommended rate, no ceiling, no `Retry-After`, no 429
  declared anywhere. Pick a conservative interval and back off on any non-2xx.
- **`create_dispatch_records=true` is destructive to your own cursor.** Once dispatched, an event
  will not come back from `PollEvents`. `GET /events/public` (`ListEvents`) with
  `was_dispatched` is the only way to look at history.
- The first-party Python SDK (`github.com/part-os/core-python`) wraps this loop as
  `paperless.listeners.BaseListener` / `OrderListener` / `QuoteListener`.
