---
name: Authenticate against a Wazo stack and scope the token
description: Exchange stack credentials for an X-Auth-Token, verify it, check it against the ACL a call needs, and revoke it when done.
api: openapi/wazo-auth-api-openapi.yml
operations: [createToken, listPolicies, createPolicies]
generated: '2026-08-17'
method: generated
source: openapi/wazo-auth-api-openapi.yml, https://wazo-platform.org/uc-doc/api_sdk/rest_api/quickstart, https://wazo-platform.org/uc-doc/api_sdk/rest_api/conventions
---

# Authenticate against a Wazo stack

Every other Wazo skill starts here. There is no public Wazo API host — you are always calling a
specific customer stack, so you need the stack hostname before anything else.

## 0. Establish the base URL

```
https://<wazo_stack>/api/auth/0.1
```

`<wazo_stack>` is the operator's own host (the docs write it as `https://<YOUR_WAZO_IP>` or
`https://mystack.example`). If you do not have it, stop and ask. Do not guess a hostname.

## 1. Mint a token — `createToken`

`POST /token` with HTTP Basic `username:password` for a `wazo-auth` user. A user created for
programmatic access should have `"purpose": "external_api"`.

```
curl -k -X POST https://<wazo_stack>/api/auth/0.1/token \
  -u "<username>:<password>" \
  -H 'Content-Type: application/json' \
  -d '{"expiration": 3600, "backend": "wazo_user"}'
```

The response carries `data.token`, `data.metadata.tenant_uuid` and `data.expires_at`.

**Ask for the shortest expiration that fits the job.** A Wazo token issued without a restricted
policy carries every permission the user has; the quickstart states plainly that such a token
"gives all permissions to anyone who knows it".

## 2. Send it on every subsequent call

```
-H 'X-Auth-Token: <token>'
```

This one header is the whole auth model for all 13 services. There is no OAuth flow, no scope
parameter, and no bearer prefix.

## 3. Know the ACL before you call

Each operation declares the permission it needs in its own description, e.g.
`confd.users.create`, `calld.calls.create`, `webhookd.subscriptions.read`. The full catalogue of
788 permission strings is in `scopes/wazo-acl-permissions.yml`.

- `HEAD /token/{token}` — is the token still valid?
- `GET /token/{token}` — token metadata, including the ACL list it carries
- `POST /token/{token}/scopes/check` — ask whether this token may perform a given ACL, **before**
  attempting the call. Prefer this to discovering a 401/403 mid-flow.

## 4. Least privilege — `listPolicies` / `createPolicies`

`GET /policies` lists ACL policies; `POST /policies` creates one. Give an automation its own
policy with the narrowest ACL set it needs (wildcards: `*` matches one segment, `#` matches the
remainder) rather than reusing an administrator token.

## 5. Revoke

`DELETE /token/{token}` when the job is finished. Do not let a long-lived all-permissions token
sit in an agent's memory.

## Failure handling

- `401` — missing/expired token, or the token lacks the ACL. Re-mint; do not retry the same token.
- `503` — a dependency is down. Retry with backoff.
- There is **no** `Retry-After` and **no** rate-limit header anywhere in Wazo. Choose your own
  backoff; see `rate-limits/wazo-rate-limits.yml`.
- **No idempotency.** Wazo publishes no `Idempotency-Key`. A retried `POST` may create a second
  resource. Before retrying a create, `GET` the collection and check whether the first attempt
  landed.
