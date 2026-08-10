---
name: Subscribe to Convelio shipment events
description: Register webhook subscriptions for Convelio's five events, verify the HMAC signature on delivery, and use the events to track a shipment the REST API gives you no read path for.
api: openapi/convelio-shipping-openapi.yml
operations:
  - createWebhook
  - listWebhooks
  - getWebhook
  - updateWebhook
  - deleteWebhook
generated: '2026-08-09'
method: generated
source: openapi/convelio-shipping-openapi.yml
---

# Subscribe to Convelio shipment events

Webhooks are not optional decoration on this API — they are the **only** way to observe
a shipment. There is no `getOrder`, no `listOrders`, and shipment status appears on no
resource. If you do not subscribe, an order you create is invisible to you after the
201.

## Register a subscription (`createWebhook`)

`POST /webhook` with a `webhook` body:

| Field | Required | Notes |
|---|---|---|
| `url` | yes | your HTTPS endpoint |
| `triggering_event_name` | yes | exactly one of the five events below |

**One event per registration.** To receive all five events you create five webhooks.
The response carries a generated `id` (UUID) and `creation_date`.

## The five events

| `triggering_event_name` | Fires when | Payload |
|---|---|---|
| `custom_quote_ready` | a price is available for a custom quote | `quote_id` |
| `quote_paid` | a quote was paid | `quote_id`, `tracking_link` |
| `order_created` | an order was created through the API | `quote_id`, `order_id` |
| `shipment_status_changed` | a new shipment status was published | `order_id`, `status` |
| `document_ready` | a new document is available | `order_id`, `document_type`, `dashboard_order_link` |

Every delivery uses the same envelope: `event` (the name), `created` (RFC 3339
timestamp), and `payload`.

`shipment_status_changed` walks a fixed lifecycle: `shipment_created` → `picked_up` →
`packing_in_progress` → `export_in_progress` → `freight_in_transit` →
`import_in_progress` → `out_for_delivery` → `shipment_completed`, with `on_hold` and
`canceled` as off-path terminals.

## Verify every delivery

Convelio signs the request body with **HMAC-SHA256**, keyed with your account API secret
token, and sends the result in the `X-Convelio-signature` header.

1. Compute HMAC-SHA256 over the **raw** request body — not a re-serialized parse — using
   your API secret as the key.
2. Compare against `X-Convelio-signature` in constant time.
3. Reject on mismatch. An unverified webhook is an unauthenticated `order_id` from the
   open internet.

Respond **`204 No Content`** on success.

## Operational realities to design around

- **Retries and ordering are not documented.** Assume at-least-once delivery and possible
  reordering. Make your handler idempotent on `(event, payload id, created)`, and never
  assume `shipment_status_changed` arrives in lifecycle order.
- **There is no replay or delivery-log endpoint.** A missed event is gone. If your
  endpoint was down, reconcile from your own records rather than asking Convelio to
  redeliver.
- **Signing secret rotation is undocumented.** Rotating your API key changes the webhook
  signing key too, since they are the same secret — plan for a verification window that
  accepts both during a rotation.

## Manage subscriptions

- `GET /webhook` (`listWebhooks`) — all registrations for the account. Returns an
  unbounded array; there is no pagination, and there are no filter parameters.
- `GET /webhook/{webhookId}` (`getWebhook`)
- `PUT /webhook/{webhookId}` (`updateWebhook`) — change the `url` or the
  `triggering_event_name`
- `DELETE /webhook/{webhookId}` (`deleteWebhook`) — returns `204`

## Test it

Register your webhooks against the sandbox host
(`https://api.sandbox.convelio.com/v2`) with a `sk_test_` key first. Note that Convelio
publishes **no trigger or replay tooling**, so there is no supported way to fire a test
delivery on demand — you exercise the path by driving a real sandbox quote and order
through `createShippingQuote` and `createShippingOrder`.

Full event catalogue: `asyncapi/convelio-webhooks.yml`.
