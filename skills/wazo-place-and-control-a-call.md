---
name: Place and control a call on a Wazo stack
description: Originate a call, list and inspect live calls, initiate a transfer, and hang up — using wazo-calld, whose operations have no operationIds.
api: openapi/wazo-calld-api-openapi.yml
operations: []
generated: '2026-08-17'
method: generated
source: openapi/wazo-calld-api-openapi.yml
---

# Place and control a call

`wazo-calld` is the runtime call-control surface, driving Asterisk over ARI/Stasis.
Base: `https://<wazo_stack>/api/calld/1.0` · Header: `X-Auth-Token`.

> **Bind by method + path, not by operationId.** 126 of the 128 operations in the calld contract
> have **no `operationId`**. There is nothing to reference, so every step below names the HTTP
> method and path verbatim from `openapi/wazo-calld-api-openapi.yml`. Do not invent an
> operationId for these; a generated client will have named them itself.

## Originate

- `POST /calls` — tenant-level originate. Body carries `source` (`user` uuid) and `destination`
  (`extension`, `context`, optionally `priority`). ACL: `calld.calls.create`.
- `POST /users/me/calls` — originate as the authenticated user. Prefer this for user-facing
  automation: it needs only `calld.users.me.calls.create` and cannot reach another tenant.

Response carries the `call_id` — keep it; every later step is keyed on it.

## Observe

- `GET /calls` — all live calls in the tenant; `GET /users/me/calls` — the caller's own.
- `GET /calls/{call_id}` — one call's live state.
- For state changes, do **not** poll. Subscribe to `call_created`, `call_answered`, `call_updated`,
  `call_ended` (see `skills/wazo-subscribe-to-events.md`); calld emits 66 events.

## Transfer

- `POST /transfers` — tenant-level; `POST /users/me/transfers` — as the user. A transfer has its
  own id and lifecycle (`transfer_created`, `transfer_answered`, `transfer_completed`,
  `transfer_abandoned`, `transfer_cancelled`).
- `GET /users/me/transfers` lists the caller's transfers.

## Hang up

`DELETE /calls/{call_id}`.

## Failure handling

- `503` is the signature calld failure and it means **Asterisk/ARI is unreachable**, not that your
  request was wrong — 195 of the 932 operations across Wazo declare a 503. Retry with backoff.
- `404` on `/calls/{call_id}` after a hangup is normal: the call is gone, not missing.
- **No idempotency.** A retried `POST /calls` places a **second real phone call**. This is the
  highest-consequence non-idempotent operation on the platform. Before retrying an originate,
  `GET /calls` and look for a call matching your source and destination.
