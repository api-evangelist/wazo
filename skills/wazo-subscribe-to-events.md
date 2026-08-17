---
name: Subscribe to Wazo platform events as webhooks
description: Register an HTTP webhook against named Wazo bus events, filter it to one user or one stack, and read the delivery log when it fails.
api: openapi/wazo-webhookd-api-openapi.yml
operations: [create_subscription, list_subscriptions, get_subscription, update_subscription, delete_subscription, create_user_subscription, list_user_subscriptions, getSubscriptionsServices]
generated: '2026-08-17'
method: generated
source: openapi/wazo-webhookd-api-openapi.yml, https://api.wazo.io/documentation/overview/webhook.html, asyncapi/wazo-events-webhooks.yml
---

# Subscribe to Wazo platform events

`wazo-webhookd` sits on the internal RabbitMQ bus and relays named events to an HTTP endpoint you
own. Base: `https://<wazo_stack>/api/webhookd/1.0`.

## 1. Pick the event names

There are **327** named events across confd, calld, agentd, dird, amid, sysconfd and webhookd.
The full catalogue — event name, owning service, routing key and required ACL — is in
`asyncapi/wazo-events-webhooks.yml`. Frequently used: `call_created`, `call_answered`,
`call_ended`, `user_created`, `user_edited`, `user_deleted`, `agent_paused`, `agent_unpaused`,
`chat_message_created`.

**Wazo publishes no AsyncAPI document.** Event names are exact strings — take them from the
catalogue, never invent one, because an unknown name subscribes successfully and then simply never
fires.

## 2. Check the relay services — `getSubscriptionsServices`

`GET /subscriptions/services`. `http` is the built-in service; anything else must be installed as
a plugin on that stack.

## 3. Create the subscription — `create_subscription`

`POST /subscriptions`. Required: `name`, `service`, `config`, `events`.

```json
{
  "name": "crm-call-sync",
  "service": "http",
  "events": ["call_created", "call_ended"],
  "config": {
    "url": "https://example.org/hooks/wazo",
    "method": "post",
    "content_type": "application/json"
  }
}
```

Narrow the firehose with the two documented filters:

- `events_user_uuid` — only events concerning that user (not compatible with every event; see
  https://wazo-platform.org/uc-doc/api_sdk/rest_api/webhookd/user_filter)
- `events_wazo_uuid` — only events from that stack

ACL: `webhookd.subscriptions.create`. For a subscription owned by the calling user instead of the
tenant, use `create_user_subscription` (`POST /users/me/subscriptions`).

## 4. Manage

`list_subscriptions` (`GET /subscriptions`, `{total, items}`), `get_subscription`,
`update_subscription` (`PUT`), `delete_subscription` (`DELETE`).

## 5. Debug delivery

`GET /subscriptions/{subscription_uuid}/logs` returns per-attempt logs — the first place to look
when your endpoint is not receiving. Wazo does not publish a retry/backoff policy for relays, so
treat delivery as at-least-once and de-duplicate on your side.

## Alternative: live stream instead of webhooks

For an in-process consumer, connect to `wss://<wazo_stack>/api/websocketd/` and subscribe to the
same event names — same catalogue, no public endpoint required. See
https://api.wazo.io/documentation/overview/websocket.html.
