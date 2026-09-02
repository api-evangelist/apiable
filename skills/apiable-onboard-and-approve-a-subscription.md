---
name: Onboard and approve an Apiable subscription
description: >-
  Create a subscription to a plan for a consumer team on an Apiable portal, approve it, and confirm
  the access it grants — the core partner-onboarding flow of the Apiable Platform API.
api: openapi/apiable-platform-api-openapi.json
operations:
  - findAllProducts
  - findAllPlans
  - findPlanById_1
  - findAllTeams
  - createSubscription
  - approveSubscription
  - rejectSubscription
  - findSubscriptionById
  - cancelAndRevokeSubscription
---

# Onboard and approve an Apiable subscription

## Before you act

- **Get a token.** POST your client id and secret to `<portal-host>/api/oauth2/token` using the
  OAuth 2.0 client-credentials grant, then send the JWT as `Authorization: Bearer <token>` on every
  call. The published spec names `https://developer.apiable.io` in `servers[]` because that is
  Apiable's own reference portal; **each customer calls their own portal host** — read the host from
  the account you are acting for, do not assume `developer.apiable.io`.
- **Scope.** Every operation in this skill requires `apiable/platform`. `apiable/cicd` is not
  enough for any of them.
- **Version.** You may pin `X-API-Version: 2024-09-25`. Omitting the header means "latest", which is
  a moving target — pin it for anything you run unattended.
- **There is no idempotency key.** `createSubscription` is a POST with no `Idempotency-Key` header
  and no documented retry-safety contract. If a call times out, **read before you retry** with
  `findAllSubscriptions` — a blind retry can create a second subscription.

## Steps

1. **Find the plan.** `findAllProducts` (`GET /api/products`) lists the catalog; `findAllPlans`
   (`GET /api/plans`) lists the plans developers can subscribe to. Both paginate with `page`,
   `size` (max 100), `sort` and `search`. Read the one you want with `findPlanById_1`
   (`GET /api/plans/{id}`) and check `status`, `securityLevel`, `numberOfAllowedSubscriptions` and
   `monetization` before subscribing anything to it.
2. **Find the team.** `findAllTeams` (`GET /api/teams`). A subscription belongs to a team, with one
   user as its primary owner and billing responsible.
3. **Create the subscription.** `createSubscription` (`POST /api/subscriptions`). Capture the
   returned `id` — every later step in this skill and every webhook payload keys on it.
4. **Approve or reject.** A subscription that needs approval sits in a pending state.
   `approveSubscription` (`POST /api/subscriptions/{id}/approve`) grants it;
   `rejectSubscription` (`POST /api/subscriptions/{id}/reject`) is the reversal. Both return
   **400** when the subscription is not in a state that allows the transition — read the state with
   `findSubscriptionById` first rather than treating 400 as a transient error.
5. **Confirm.** `findSubscriptionById` (`GET /api/subscriptions/{id}`) returns `status`, the `plan`,
   the `team`, the `owner`, and `auth` — the credential the consumer will use.

## Undoing it

`cancelAndRevokeSubscription` (`DELETE /api/subscriptions/{id}`) cancels in the billing system and
revokes API access. It takes a body of `{"cancelAt": <unix seconds UTC>}` — **you choose when the
cancellation lands.** Apiable publishes no deadline after which a subscription can no longer be
cancelled, and **no un-cancel operation**: once it is cancelled you create a new subscription, you
do not restore the old one. Treat cancellation as one-way.

## Errors

The API returns plain HTTP status codes with a JSON body carrying a message. It is **not** RFC 9457
and there is no error-code registry. Only 400, 401 and 404 are declared anywhere in the contract.

- **401** — every operation declares it; your token is missing, expired, or lacks `apiable/platform`.
- **404** — the plan, team or subscription id does not exist on this portal.
- **400** — invalid input, or a state transition the subscription does not allow.
- **429 and 5xx are not declared.** Apiable publishes no rate limits and no `Retry-After` header, so
  back off on your own schedule and do not expect a retry hint.
