---
title: Plans, checkout and payment
description: List the plans on sale, start a checkout on Moyasar, and see how a verified payment changes the plan. Prices in SAR per month, before VAT.
icon: "credit-card"
---

segmnt bills in SAR, per month. This page shows how to list the plans, how to
send a customer to the checkout, and how a confirmed payment changes the plan.

This guide is written in simple English. It is in English only; the Arabic
version is on the Arabic docs site at
[test-docs.segmnt.cloud](https://test-docs.segmnt.cloud/ar/billing/).

## Before you start

- API in TEST: `https://test-api.segmnt.cloud`
- Every billing call on this page uses your **session cookie** (the dashboard
  session). An API key cannot call them.
- Prices are in **SAR per month, before VAT**.
- The plans on sale are **Starter**, **Pro** and **Agency**. The free plan is
  the default; you do not buy it.

## 1. The plans on sale

`GET /v1/billing/plans` returns the plans a customer can buy.

```bash
curl "$BASE/v1/billing/plans" \
  -H "Cookie: segmnt_session=$SEGMNT_SESSION"
```

```json
{
  "data": [
    {
      "id": "starter",
      "name": "Starter",
      "price": { "amount": 5900, "currency": "SAR", "interval": "month" },
      "limits": {
        "storedSeconds": 90000,
        "uploadSecondsPerPeriod": 18000,
        "watchedMinutesPerPeriod": 20000,
        "verifiedDomains": 1,
        "projects": 1,
        "maxQuality": 1080,
        "whiteLabel": false
      }
    },
    {
      "id": "pro",
      "name": "Pro",
      "price": { "amount": 34900, "currency": "SAR", "interval": "month" },
      "limits": {
        "storedSeconds": 360000,
        "uploadSecondsPerPeriod": 54000,
        "watchedMinutesPerPeriod": 250000,
        "verifiedDomains": 10,
        "projects": 3,
        "maxQuality": 2160,
        "whiteLabel": true
      }
    },
    {
      "id": "agency",
      "name": "Agency",
      "price": { "amount": 129900, "currency": "SAR", "interval": "month" },
      "limits": {
        "storedSeconds": 1440000,
        "uploadSecondsPerPeriod": 108000,
        "watchedMinutesPerPeriod": 1000000,
        "verifiedDomains": 50,
        "projects": 25,
        "maxQuality": 2160,
        "whiteLabel": true
      }
    }
  ]
}
```

The same prices, in SAR:

| Plan | `id` | Price (SAR / month) |
| --- | --- | --- |
| Starter | `starter` | 59 |
| Pro | `pro` | 349 |
| Agency | `agency` | 1,299 |

Notes:

- `price.amount` is in **halalas** (1 SAR = 100 halalas). `5900` halalas is
  59 SAR. This is the same unit the API uses everywhere.
- `price.currency` is `SAR` and `price.interval` is `month`.
- `price` is **before VAT**. See [VAT](#vat) below.
- `limits` are the plan limits (stored hours, new uploads, watched minutes,
  domains, projects, quality and the logo). See
  [Usage and plan limits](/usage-and-limits) for what they mean.
- The prices and limits come from the plan catalog. They are the same
  numbers you see on the site.
- This call needs the session cookie. Without it you get `401 unauthorized`.

## 2. Start a checkout

`POST /v1/billing/checkout` returns the payment provider's hosted page. Send the
customer there. The customer pays on **Moyasar** with mada, Apple Pay, STC Pay
or a card.

```bash
curl -X POST "$BASE/v1/billing/checkout" \
  -H "Cookie: segmnt_session=$SEGMNT_SESSION" \
  -H "Content-Type: application/json" \
  -d '{"planId":"pro"}'
```

Answer (`201`):

```json
{
  "id": "inv_5f0b2c8a",
  "checkoutUrl": "https://checkout.moyasar.com/invoice/inv_5f0b2c8a",
  "expiresAt": null
}
```

Notes:

- **The plan does not change here.** It changes only after the provider
  confirms the payment (section 3).
- `checkoutUrl` is the provider's hosted page. Redirect the user to it.
- `expiresAt` is `null` when the provider sets no deadline. Moyasar invoices do
  not expire by default, so do not invent one.
- `planId` must be one of the plans on sale. An unknown or free plan gets
  `400 invalid_request`.
- A `discountCode` is refused with `400 invalid_request` while discount codes
  are not built. We refuse it instead of charging the wrong price.
- If online payment is not configured, checkout gets
  `503 billing_unavailable`. Use a manual invoice.
- If the provider does not answer, checkout gets
  `502 billing_provider_error`. Try again.

## 3. The payment that changes the plan

When the customer pays, Moyasar calls
`POST /v1/billing/providers/moyasar/webhook`. This route is for the provider,
not for customers. It has no segmnt session; it is checked with the provider's
own secret token.

What happens on every verified event:

1. The provider's secret token is checked with a constant-time compare.
2. The webhook body is **never trusted**. We read the merchant invoice back
   from the provider API.
3. The amount and currency are checked against the plan's fixed price
   (plus 15% VAT when VAT mode is on).
4. The payment, the 15% VAT invoice, the subscription and the account plan are
   written in **one batch**, so the event is applied once and cannot be applied
   half way.

This makes the webhook **idempotent**: a provider retries a webhook, so the
same event can arrive more than once. One provider invoice is recorded at most
once, a second delivery answers `204` and does not change the plan again or
extend the period.

A **refund** reverses an applied payment. When the provider reports a refunded
payment, the payment is recorded as `refunded`, the subscription is set to
`canceled`, and the account returns to the **free** plan. The invoice row is
**kept**: a refund is not a delete, and the fiscal sequence must stay gapless
(a ZATCA credit note is a later open point). A refund is idempotent too, so
a retried refund changes nothing.

Answers:

| Status | When |
| --- | --- |
| `204` | Applied, already applied, reversed by a refund, or not a paid payment. A refund is accepted so the provider does not retry. |
| `400` `billing_mismatch` | A verified payment that does not match what was ordered. No plan change. |
| `401` `invalid_signature` | The secret token is missing or wrong, or there is no invoice to reconcile. |
| `404` `not_found` | Unknown payment provider. |
| `502` `billing_provider_error` | The provider API could not be read. The provider will retry. |
| `503` `billing_unavailable` | Webhooks are not configured. |

A verified event that does not match is refused with `400` and changes no plan.
This covers a wrong amount, a currency that is not SAR, a missing plan or
account in the invoice, an unknown plan, and an account we do not have.

## 4. Check the plan after payment

After a confirmed payment, `GET /v1/billing/plan` returns the paid plan, an
active subscription period, the new limits and your usage. The call is
documented on the [Usage and plan limits](/usage-and-limits) page.

```bash
curl "$BASE/v1/billing/plan" \
  -H "Cookie: segmnt_session=$SEGMNT_SESSION"
```

```json
{
  "plan": { "id": "pro", "name": "Pro" },
  "status": "active",
  "currentPeriod": {
    "start": "2026-10-14T10:00:00.000Z",
    "end": "2026-11-14T10:00:00.000Z"
  },
  "limits": {
    "storedSeconds": 360000,
    "watchedMinutesPerPeriod": 250000,
    "verifiedDomains": 10,
    "maxQuality": 2160,
    "whiteLabel": true
  },
  "usage": { "storedSeconds": 5400, "watchedMinutes": 842, "verifiedDomains": 1 }
}
```

Notes:

- `limits` is the paid plan's limits (stored hours, watched minutes, domains,
  quality and the logo). In `limits`, `null` means **no limit**.
- `usage` is your account now: stored seconds, watched minutes in the current
  period, and verified domains.
- The new-upload and project limits are enforced, but they are not in this
  answer yet. See [Usage and plan limits](/usage-and-limits) for the full
  limits table.

The paid period is one calendar month from the payment, in UTC. A shorter month
is clamped, so 31 January becomes 28 or 29 February.

## VAT

VAT is not on yet. Today VAT is **off**:

- When VAT is off, the customer pays the plan price and the invoice shows
  `0` VAT.
- When VAT mode is **on**, the amount is the plan price **plus 15%**, and the
  invoice records the VAT.

The webhook checks the paid amount with the same rule, so a payment that does
not match the expected amount is refused. VAT is ZATCA's money, not segmnt
revenue.

The stored invoice **lines** carry the VAT: each line has `unitAmount` (net),
`vatAmount` and `amount` (gross, net + VAT). So the lines add up to
`invoices.total` when VAT mode is on.

## Not live yet

- **Discount codes.** Checkout refuses a code instead of overcharging.
- **The annual plan.** Only the monthly plan is on sale.
- **The invoice PDF.** The 15% VAT invoice is written, but its PDF file is not
  built yet.

## Errors

Errors use `application/problem+json`. Use the `code` field in your code, not
the text. The `type` field links to `https://docs.segmnt.cloud/errors/{code}`.

| Status | `code` | Meaning | What to do |
| --- | --- | --- | --- |
| 400 | `invalid_request` | Unknown `planId`, or a `discountCode` while codes are not live | Send a plan on sale, and no code. |
| 400 | `billing_mismatch` | Provider webhook: the payment does not match the order (amount, currency, plan or account) | Check the invoice in the provider dashboard. No plan change was made. |
| 401 | `unauthorized` | Missing, wrong or expired session | Sign in again. |
| 401 | `invalid_signature` | Provider webhook: the secret token is wrong, or there is no invoice | Check the provider's webhook secret. |
| 404 | `not_found` | Provider webhook: unknown payment provider | Use the provider's webhook URL (`/v1/billing/providers/moyasar/webhook`). |
| 405 | `method_not_allowed` | Wrong method for the URL | See the method in each section above. |
| 502 | `billing_provider_error` | The payment provider did not answer | Try again. For a webhook, the provider retries. |
| 503 | `billing_unavailable` | Online payment or webhooks are not configured | Use a manual invoice, or ask us to configure the provider. |

## Tested in TEST

Checked against `https://test-api.segmnt.cloud` on 14 Oct 2026:

- `GET /v1/billing/plans` with no session -> `401 unauthorized`.
- `POST /v1/billing/checkout` with no session -> `401 unauthorized`.
- `POST /v1/billing/providers/moyasar/webhook` with no webhook secret
  configured -> `503 billing_unavailable`.
- `POST /v1/billing/checkout` with an unknown plan or a `discountCode` ->
  `400 invalid_request` (unit test).

The paid path (a verified invoice applies the plan, the 15% VAT invoice,
sequential invoice numbers, idempotent replay, and every `400` refusal path) is
proved by the API tests in `api/src/billing.test.ts`. The live Moyasar sandbox
run needs the payment provider's sandbox keys, so the end-to-end provider call is
not run on TEST yet.
