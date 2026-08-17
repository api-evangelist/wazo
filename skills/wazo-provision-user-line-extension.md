---
name: Provision a Wazo user with a line, SIP endpoint and extension
description: Walk the confd association chain — user, line, SIP endpoint, extension — in the order Wazo requires, and unwind it safely on failure.
api: openapi/wazo-confd-api-openapi.yml
operations: [create_user, create_line, create_endpoint_sip, associate_line_endpoint_sip, create_extension, associate_line_extension, associate_user_line, list_user, dissociate_user_line, dissociate_line_extension, dissociate_line_endpoint_sip]
generated: '2026-08-17'
method: generated
source: openapi/wazo-confd-api-openapi.yml, https://wazo-platform.org/uc-doc/api_sdk/rest_api/examples
---

# Provision a user with a line, SIP endpoint and extension

The single most common Wazo automation. `wazo-confd` models a phone user as four separate
resources joined by explicit association calls — creating them is not enough, you must associate
them, and the order matters.

Base: `https://<wazo_stack>/api/confd/1.1` · Header: `X-Auth-Token` (see
`skills/wazo-authenticate-and-scope-token.md`).

## Steps

1. **`create_user`** — `POST /users` with `firstname`, `lastname`. Keep the returned `uuid` **and**
   `id`; confd uses the integer `id` in association paths and the `uuid` almost everywhere else.
   ACL: `confd.users.create`.
2. **`create_endpoint_sip`** — `POST /endpoints/sip`. Keep `uuid`. ACL:
   `confd.endpoints.sip.create`.
3. **`create_line`** — `POST /lines` with the target `context`. Keep `id`. ACL:
   `confd.lines.create`.
4. **`associate_line_endpoint_sip`** — `PUT /lines/{line_id}/endpoints/sip/{sip_uuid}`. Empty body.
5. **`create_extension`** — `POST /extensions` with `exten` and `context`. Keep `id`. ACL:
   `confd.extensions.create`.
6. **`associate_line_extension`** — `PUT /lines/{line_id}/extensions/{extension_id}`. Empty body.
   (`create_line_extension`, `POST /lines/{line_id}/extensions`, creates and associates in one
   call — use it only when you do not need the extension to exist independently.)
7. **`associate_user_line`** — `PUT /users/{user_id}/lines/{line_id}`. This is the step that makes
   the phone ring for that user. `associate_user_lines` (`PUT /users/{user_id}/lines`) replaces the
   whole ordered list at once — it will **remove** associations you omit.

## Verify

`list_user` (`GET /users?search=<lastname>`) returns `{total, items}`. Fetch the user and confirm
`lines[]` is populated.

## Unwind on failure — this matters

Wazo has **no idempotency key and no transaction**. A partial run leaves orphans. Undo in reverse:
`dissociate_user_line` → `dissociate_line_extension` → `dissociate_line_endpoint_sip` → delete the
extension, line, SIP endpoint and user. Never blind-retry step 1: check
`GET /users?search=` first, or you will create a duplicate user.

## Conventions

- Pagination on every list: `limit`, `offset`, `order`, `direction`, `search`; response
  `{total, items}`.
- Multi-tenant: send `Wazo-Tenant: <tenant_uuid>` to act inside a sub-tenant; add `recurse=true`
  to list across sub-tenants.
- `409` means the resource already exists or you violated an association constraint — read the
  message list; confd returns errors as a JSON **list** even for one message.
