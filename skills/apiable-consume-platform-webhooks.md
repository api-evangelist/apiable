---
name: Consume and verify Apiable platform webhooks
description: >-
  Register an outbound webhook on an Apiable portal, verify every delivery's Standard Webhooks
  signature, and react to subscription, invoice and scope-grant events without polling.
api: openapi/apiable-platform-api-openapi.json
operations:
  - registerWebhook
  - findAllWebhooks
  - getWebhookById
  - updateWebhook
  - testWebhookById
  - getWebhookHistory
  - unregisterWebhook
  - findSubscriptionById
---

# Consume and verify Apiable platform webhooks

## Before you act

- **Scope.** `apiable/cicd`, `apiable/platform` or `apiable/admin` all authorize webhook
  management.
- **You need an HTTPS endpoint** that accepts a POST and returns 200, 201, 202 or 204. Anything
  else counts as a failure.

## Steps

1. **Register.** `registerWebhook` (`POST /api/webhooks`) with `url` and `events`. Leave `whsec`
   out and let Apiable generate the signing secret — if you set your own it must be base64-encoded,
   prefixed `whsec_`, and decode to 24–64 bytes or the request is rejected with a 400. Use
   `headers` for anything extra your endpoint checks; the `authorization` field is deprecated.
2. **Subscribe to the right events.** Nine types exist:
   `SUBSCRIPTION_CREATED`, `SUBSCRIPTION_CANCELLED`, `SUBSCRIPTION_CHANGED`,
   `SUBSCRIPTION_AUTH_CHANGED`, `INVOICE_ATTENTION_REQUIRED`, `SCOPE_GRANT_REQUESTED`,
   `SCOPE_GRANT_APPROVED`, `SCOPE_GRANT_DECLINED`, `SCOPE_GRANT_REVOKED`.
   **Watch this divergence:** the published OpenAPI's `WebhookConf` schema enumerates only the
   first five. The four `SCOPE_GRANT_*` types are documented but absent from the contract's enum,
   so a strict client generated from the spec will reject them. Validate loosely on the event type.
3. **Verify every delivery.** Apiable signs with the Standard Webhooks scheme and sends
   `webhook-id`, `webhook-timestamp` and `webhook-signature`. Build the signed content as
   `{webhook-id}.{webhook-timestamp}.{raw-body}`, HMAC-SHA256 it with the base64-decoded secret
   (after stripping `whsec_`), base64-encode the result, and compare against the `v1,<base64>`
   entry. Verify against the **raw** body, never a re-serialized one. Reject anything whose
   `webhook-timestamp` is more than 5 minutes from now. Prefer an off-the-shelf Standard Webhooks
   library over hand-rolling this.
4. **Act on the event.** The payload carries `id`, `type`, `timestamp`, `data` and an optional
   `source`. For a subscription event, call `findSubscriptionById`
   (`GET /api/subscriptions/{id}`) to read the current state rather than trusting the payload
   snapshot — this is the pattern Apiable's own docs recommend, and it removes polling entirely.
5. **Prove it works before you rely on it.** `testWebhookById`
   (`GET /api/webhooks/{id}/test`) sends a signed `TEST_EVENT` and returns the status and body
   Apiable received. This is the only rehearsal facility the API offers — use it in staging.

## When deliveries fail

`getWebhookHistory` (`GET /api/webhooks/{id}/history`) returns the last **24 hours** of delivery
attempts with the status and body Apiable recorded. Delivery states are `PENDING`, `SUCCESS`,
`ERROR` and `SKIPPED`. A failed delivery retries **hourly, up to 3 attempts, then stops** — after
that the event is gone and there is no replay operation. If your endpoint was down for more than
three hours, reconcile by reading state through the API; do not wait for a redelivery that will
never come.

## Undoing it

`unregisterWebhook` (`DELETE /api/webhooks/{id}`) removes the registration. Deliveries already
dispatched are not recalled. `updateWebhook` (`PUT /api/webhooks/{id}`) changes url, events,
secret or headers in place — prefer it over delete-and-recreate, which loses the delivery history.
