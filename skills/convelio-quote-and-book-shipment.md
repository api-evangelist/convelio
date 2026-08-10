---
name: Quote and book a fine art shipment with Convelio
description: Price a fine art shipment, create a quote, handle the instant-price and manual-pricing branches, and convert the quote into a booked order.
api: openapi/convelio-shipping-openapi.yml
operations:
  - estimateShippingPrice
  - createShippingQuote
  - getShippingQuote
  - createShippingOrder
generated: '2026-08-09'
method: generated
source: openapi/convelio-shipping-openapi.yml
---

# Quote and book a fine art shipment

Convelio prices door-to-door fine art logistics — packing, crating, customs, freight and
white-glove delivery — as one all-inclusive number. This skill covers the whole commercial
path: estimate, quote, then order.

## Before you start

- Base URL is `https://api.convelio.com/v2` for live and `https://api.sandbox.convelio.com/v2`
  for test. **You do not choose the environment with a parameter** — the key does. A
  `sk_test_` key only authenticates against the sandbox host, a `sk_live_` key only
  against production.
- Authenticate every request with `Authorization: token <secret_key>`.
- Send `Content-Type: application/json` (or `application/vnd.convelio-shipping.v2+json`)
  and `Accept: application/json`.
- Optionally set `?currency=` to `EUR`, `USD` or `GBP`. All amounts in responses are
  positive integers in the smallest currency unit — `10000` means 100.00.
- Keys are not self-service. If you do not have one, request it from `api@convelio.com`.

## Step 1 — Estimate before you commit (`estimateShippingPrice`)

`POST /shipping/estimate/price`

Use this when you only need a number — a price band on a listing page, a feasibility
check — and do not want a record created. The request body is a `shipment-estimation`:
at minimum a `delivery` (with `type` and `address`), plus `pickups[]` and a
`shipping_speed`.

Returns `200` with a `price` object carrying `currency_code`, `vat_excluded_amount` and
`vat_included_amount`.

A `422` here is not always your fault. It carries two distinct meanings: a malformed
body (read `validation_messages`) **or** "No shipment estimation available for given
request" — Convelio cannot instantly price this shipment. Distinguish them by checking
whether `validation_messages` is present.

## Step 2 — Create the quote (`createShippingQuote`)

`POST /shipping/quote`

Required fields on the `quote-create` body: `delivery`, `shipping_speed`, `pickups`.
Each `pickup` requires an `address` and `items`; each `item` requires a `value`
(a `commercial-value` with `amount` + `currency_code`), a `current_packing`, and a
`description`. Contacts require `first_name`, `last_name`, `email` and `phone`.

Returns `201` with a `quote` — but **read the `status` field before you do anything
else**:

- `created` — Convelio priced it instantly. `price` is populated. Go to step 4.
- `processing` — Convelio could not price instantly (unusual geography, oversized item,
  high commercial value). An operations team produces a custom quote **within 24 hours**.
  `price` is not usable yet. Go to step 3.

Treating `processing` as a failure is the most common integration mistake against this
API. It is a normal, expected outcome.

## Step 3 — Wait out a `processing` quote

Two ways to learn the price landed:

1. **Preferred — subscribe to the `custom_quote_ready` webhook** (see the
   `convelio-subscribe-to-shipment-events` skill). Its payload carries the `quote_id`.
2. **Polling — `getShippingQuote`**, `GET /shipping/quote/{quoteId}`, until `status`
   becomes `created`. There is no documented rate limit, so poll conservatively — no
   faster than once every few minutes against a 24-hour SLA-less window.

## Step 4 — Book the shipment (`createShippingOrder`)

`POST /shipping/order`

The body is an `order-create`: `quote_id` is **required**, plus `billing_details`
(`name`, `email`, `phone`, `address`). This is the only id-reference relationship in the
API — an order always descends from a quote.

Returns `201` with the `order`.

## Retry safety — read this before you write a retry

**Neither `createShippingQuote` nor `createShippingOrder` is idempotent.** Convelio
publishes no `Idempotency-Key` header and performs no server-side dedupe. A blind retry
after a network timeout can create a duplicate quote or, worse, **double-book and
double-bill a physical fine art shipment**.

Handle it on your side:

- Do not auto-retry a `POST /shipping/order` that timed out. Treat the outcome as
  unknown.
- Reconcile instead: subscribe to `order_created` (payload carries both `quote_id` and
  `order_id`) and use your own `quote_id` as the dedupe key — one order per quote.
- Retry freely on `estimateShippingPrice` and `getShippingQuote`; both are safe reads.

## Errors

All errors return an RFC 7807-shaped body (`type`, `title`, `status`, `detail`) served
as `application/json`.

| Status | Meaning | What to do |
|---|---|---|
| 400 | The URL requested is not valid | Fix the path |
| 401 | Token not found | Check the `Authorization: token <key>` form, and that the key mode matches the host |
| 403 | Authenticated but not allowed | Contact api@convelio.com |
| 404 | Not found | Check the `quoteId` |
| 422 | Unprocessable | Read `validation_messages`; on estimate, may mean "not instantly priceable" |
| 500 | Internal error | Retry safe reads only |

Do not branch on the `type` URI — Convelio returns a generic W3C status-code URL rather
than a per-problem identifier, so `type` does not distinguish two 422s.

Full catalogue: `errors/convelio-problem-types.yml`.
