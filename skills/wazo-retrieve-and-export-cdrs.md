---
name: Retrieve and export Wazo call detail records
description: Query CDRs for a tenant or the authenticated user, pull recording media, and drive an asynchronous export to ZIP.
api: openapi/wazo-call-logd-api-openapi.yml
operations: []
generated: '2026-08-17'
method: generated
source: openapi/wazo-call-logd-api-openapi.yml
---

# Retrieve and export call detail records

`wazo-call-logd` is the reporting surface. Base: `https://<wazo_stack>/api/call-logd/1.0` ·
Header: `X-Auth-Token`.

> Like calld, most call-logd operations carry **no `operationId`** (19 of 22). Bind by method +
> path from `openapi/wazo-call-logd-api-openapi.yml`.

## Query

- `GET /cdr` — tenant CDRs. `GET /users/me/cdr` — the authenticated user's own.
  ACL: `call-logd.cdr.read` / `call-logd.users.me.cdr.read`.
- `GET /cdr/{cdr_id}` — one record.
- Filters and paging follow the platform convention: `from`, `until`, `limit`, `offset`, `order`,
  `direction`, `search`; the response is `{total, items}`. Page with `offset`, and always read
  `total` rather than probing until an empty page.
- Request `Accept: text/csv` on `GET /cdr` for a CSV stream instead of JSON — cheaper than paging
  JSON for a bulk pull.

## Recordings

- `GET /cdr/{cdr_id}/recordings/{recording_uuid}/media` — download the audio.
- `DELETE /cdr/{cdr_id}/recordings/{recording_uuid}/media` — **destructive and not reversible.**
  Confirm with a human before calling it; there is no undo and no idempotency guard.

## Asynchronous export

Large exports are a job, not a response body:

1. Start the export from the CDR endpoint with the export parameters.
2. `GET /exports/{export_uuid}` — poll `status` until it is finished.
3. `GET /exports/{export_uuid}/download` — retrieve the ZIP.

## Contact-centre statistics

The Support Centre endpoints under `/queues/.../statistics` and `/agents/.../statistics` return
aggregated queue and agent performance. Retention behaviour is configured on
`/retention` — read it before assuming a historical window exists.

## Failure handling

- `404` on an export uuid means it expired — restart the export rather than retrying the download.
- No rate limits are published; if you are walking a long date range, throttle yourself.
