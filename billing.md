---
title: Plans, checkout and payment
description: List the plans on sale, start a checkout with an optional discount code, and see how a verified payment changes the plan. Prices in SAR per month, before VAT.
icon: "credit-card"
---

segmnt bills in SAR, per month. This page shows how to list the plans, how to
send a customer to the checkout (with an optional discount code), and how a
confirmed payment changes the plan.

This guide is written in simple English.

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
customer there. When `BILLING_PROVIDER=polar`, the customer pays on **Polar**,
our payment provider. Polar is the **Merchant of Record**: Polar sells to the
customer and issues the invoice. Without this setting, checkout still uses
Moyasar.

```bash
curl -X POST "$BASE/v1/billing/checkout" \
  -H "Cookie: segmnt_session=$SEGMNT_SESSION" \
  -H "Content-Type: application/json" \
  -d '{"planId":"pro"}'
```

Answer (`201`):

```json
{
  "id": "cs_5f0b2c8a",
  "checkoutUrl": "https://polar.sh/checkout/cs_5f0b2c8a",
  "expiresAt": null
}
```

Notes:

- **The plan does not change here.** It changes only after the provider
  confirms the payment (section 3).
- `checkoutUrl` is Polar's hosted page. Redirect the user to it. Treat it as
  opaque; its shape is Polar's, not ours.
- `expiresAt` is the deadline Polar sets on the checkout, or `null` when Polar
  sets none. Use it when it is there; do not invent one.
- `planId` must be one of the plans on sale. An unknown or free plan gets
  `400 invalid_request`.
- `discountCode` is optional, and it works on the **Moyasar** checkout only.
  With `BILLING_PROVIDER=polar`, a code is refused with `400 invalid_request`
  until the Polar discount path is built. See [Discount codes](#discount-codes)
  below.
- If online payment is not configured, checkout gets
  `503 billing_unavailable`. Use a manual invoice.
- If the provider does not answer, checkout gets
  `502 billing_provider_error`. Try again.

## Discount codes

`POST /v1/billing/checkout` takes an optional `discountCode`. A code is applied
when it has been **issued to you**; creating a code is an internal segmnt action,
so this page does not show how to create one.

Discount codes work on the **Moyasar** checkout. When `BILLING_PROVIDER=polar`,
checkout refuses a `discountCode` with `400 invalid_request` until the Polar
discount path is built, so a code can never charge the full price by accident.

```bash
curl -X POST "$BASE/v1/billing/checkout" \
  -H "Cookie: segmnt_session=$SEGMNT_SESSION" \
  -H "Content-Type: application/json" \
  -d '{"planId":"pro","discountCode":"FOUNDING-30"}'
```

- A **founding** code gives **30% off** and a **referral** code gives **10% off**.
  The **total discount is capped at 40%**, so no code can take off more than that.
- Codes are **case-insensitive** and trimmed, so `founding-30` and `FOUNDING-30 `
  are the same code.
- The code is **checked before Moyasar is called**. An unknown, expired or
  used-up code answers `400 invalid_request` with a clear message, so the
  customer is never charged the full price by accident.
- The discount is applied to the plan price **before VAT**: VAT (15%) is charged
  on the discounted net, not on the full price.
- The 15% VAT invoice shows the plan line and, when a code was applied, a second
  **negative discount line**. The lines still add up to the invoice total.
- The code is **counted only when a payment is confirmed**. A provider retry does
  not count the same code twice.
- A confirmed payment stores the code on the subscription (`discount_code`).

## 3. The payment that changes the plan

When the customer pays, Polar calls
`POST /v1/billing/providers/polar/webhook`. This route is for the provider, not
for customers. It has no segmnt session; it is checked with Polar's webhook
signature.

What happens on every verified event:

1. The Polar signature is checked before anything else. Polar signs with the
   **Standard Webhooks** scheme: the headers `webhook-id`, `webhook-timestamp`
   and `webhook-signature`, over `<id>.<timestamp>.<body>`. A timestamp more
   than 5 minutes from now is refused, so an old capture cannot be replayed.
2. The webhook body is **never trusted**. We read the payment back from the
   provider API.
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

Polar is the **Merchant of Record**. Polar sells to the customer and issues the
customer's VAT invoice. Our own `invoices` rows are internal records: we keep
them for entitlements and reporting, and we do not send a second tax invoice.

Which Polar events matter:

- `order.paid` — the first payment and every renewal. We read the order back
  from Polar and apply the plan.
- `order.refunded`, `refund.created` and `refund.updated` — we read the order
  back and reverse the applied plan.
- `subscription.canceled` and `subscription.past_due` — the customer canceled, or
  a renewal is being retried (dunning). Access **keeps running**; only the
  subscription state is recorded.
- `subscription.revoked` — the subscription ended (for example unpaid after the
  retries). The plan is taken away, like a refund but with no money returned.
- `subscription.created`, `subscription.active`, `subscription.updated` and any
  other event we do not act on are **verified and answered with `204`**, so Polar
  stops retrying. `order.paid` already applied the plan.

Answers:

| Status | When |
| --- | --- |
| `204` | Applied, already applied, reversed by a refund, or not a paid payment. A refund is accepted so the provider does not retry. |
| `400` `billing_mismatch` | A verified payment that does not match what was ordered. No plan change. |
| `401` `invalid_signature` | The Polar signature is missing or wrong. |
| `404` `not_found` | Unknown payment provider. |
| `502` `billing_provider_error` | The provider API could not be read. The provider will retry. |
| `503` `billing_unavailable` | Webhooks are not configured. |

A verified event that does not match is refused with `400` and changes no plan.
This covers a wrong amount, a currency that is not SAR, a missing plan or
account in the invoice, an unknown plan, and an account we do not have.

Discount codes and Polar discounts are different things. Our **discount codes**
work on the Moyasar checkout: the amount is reduced before the provider is called
and a negative line is added to the invoice. When `BILLING_PROVIDER=polar`,
checkout refuses a `discountCode` with `400 invalid_request` until the Polar
discount path is built, so a code never charges the full price by accident. Polar
can also report its own discount on the order; when it does, we check
`charged + discount` against the catalog price and refuse anything else with
`400`.

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

With Polar as the **Merchant of Record**, Polar issues the customer's tax
invoice. Our `invoices` rows mirror it for our records and for entitlements.

## Provider settings

These are settings for our own deployment, not calls a customer makes. They are
listed here so the docs match the API.

- `BILLING_PROVIDER=polar` switches checkout from Moyasar to Polar. The Moyasar
  adapter stays in the code, but it is not used for checkout when this is set.
- `POLAR_ACCESS_TOKEN` is the Polar API token. Checkout needs it; without it, or
  without `POLAR_PRODUCT_IDS`, checkout answers `503 billing_unavailable`.
- `POLAR_PRODUCT_IDS` maps a plan id to the Polar product id. Checkout sends the
  product id of the plan the customer chose. It is a JSON object, for example
  `{"starter":"<starter product>","pro":"<pro product>","agency":"<agency product>"}`.
- `POLAR_API_BASE` is the Polar API base. It defaults to the sandbox
  (`https://sandbox-api.polar.sh/v1`); production sets
  `https://api.polar.sh/v1`.
- `POLAR_WEBHOOK_SECRET` verifies the Polar webhook. Without it the webhook
  answers `503 billing_unavailable`.

No secret and no real product id is written on this page.

## Not live yet

- **The annual plan.** Only the monthly plan is on sale.
- **Discount codes on Polar.** They work on Moyasar. On Polar, checkout refuses a
  code until the Polar discount path is built.
- **The invoice PDF.** The 15% VAT invoice is written, but its PDF file is not
  built yet.

## Errors

Errors use `application/problem+json`. Use the `code` field in your code, not
the text. The `type` field links to `https://docs.segmnt.cloud/errors/{code}`.

| Status | `code` | Meaning | What to do |
| --- | --- | --- | --- |
| 400 | `invalid_request` | Unknown `planId`; an unknown, expired or used-up `discountCode`; or a `discountCode` while Polar is the active provider | Send a plan on sale and a valid code (or no code). With Polar, send no code. |
| 400 | `billing_mismatch` | Provider webhook: the payment does not match the order (amount, currency, plan or account) | Check the invoice in the provider dashboard. No plan change was made. |
| 401 | `unauthorized` | Missing, wrong or expired session | Sign in again. |
| 401 | `invalid_signature` | Provider webhook: the Polar signature is wrong | Check the Polar webhook secret. |
| 404 | `not_found` | Provider webhook: unknown payment provider | Use the provider's webhook URL (`/v1/billing/providers/polar/webhook`). |
| 405 | `method_not_allowed` | Wrong method for the URL | See the method in each section above. |
| 502 | `billing_provider_error` | The payment provider did not answer | Try again. For a webhook, the provider retries. |
| 503 | `billing_unavailable` | Online payment or webhooks are not configured | Use a manual invoice, or ask us to configure the provider. |

## Tested in TEST

Checked against `https://test-api.segmnt.cloud` on 14 Oct 2026:

- `GET /v1/billing/plans` with no session -> `401 unauthorized`.
- `POST /v1/billing/checkout` with no session -> `401 unauthorized`.
- `POST /v1/billing/providers/polar/webhook` with no webhook secret
  configured -> `503 billing_unavailable`.
- `POST /v1/billing/checkout` with an unknown plan -> `400 invalid_request`
  (unit test).
- `POST /v1/billing/checkout` with an unknown, expired or used-up `discountCode`
  -> `400 invalid_request` (unit test).
- `POST /v1/billing/checkout` with a `discountCode` while
  `BILLING_PROVIDER=polar` -> `400 invalid_request` (unit test).

The paid path (a verified invoice applies the plan, the 15% VAT invoice,
sequential invoice numbers, idempotent replay, and every `400` refusal path) is
proved by the API tests in `api/src/billing.test.ts`. The same tests cover the
discount paths: a founding code (30%), a referral code (10%), the 40% cap, the
plan line plus the negative discount line that still reconcile to the total, and
a replay that does not count the code twice.

The Polar paths are proved by the API tests in `api/src/billing.test.ts`: a real
signed `order.paid` applies the plan, payment, invoice and subscription; a retry
applies nothing; a discounted order is accepted when `charged + discount` is the
catalog price; a wrong amount, a non-SAR order and a wrong signature are refused;
a refund revokes the plan; `subscription.revoked` takes the plan away;
`subscription.canceled` and `subscription.past_due` keep access; and a
`subscription.active` event is answered `204` with no plan change.

The live provider runs (the Moyasar sandbox and the Polar sandbox) both need
provider keys, so the end-to-end provider calls are not run on TEST yet.
