---
title: Phone verification
description: A new free account verifies a mobile number (Saudi or GCC) with a one-time code before its first upload. Start the code, verify it, and see the upload error when verification is missing.
icon: "smartphone"
---

A new **free** account must verify a **Saudi or GCC mobile number** before it
can upload its first video. One phone number is meant to verify one account (the
API does not enforce that yet). The number is stored only as a salted hash, so
nobody can read it back.

You verify the number from your own app, with the same session cookie the
dashboard uses. The dashboard step is not built yet; see
[Not live yet](#not-live-yet).

Both calls below use your **session cookie**. An API key cannot call them.

This guide is written in simple English. It is in English only; the Arabic
version is on the Arabic docs site at
[test-docs.segmnt.cloud](https://test-docs.segmnt.cloud/ar/phone-verification/).

## Before you start

- API in TEST: `https://test-api.segmnt.cloud`
- Everything on this page uses the **session cookie** (`segmnt_session`), not an
  API key.
- The number must be a **Saudi or GCC mobile**. Send a full international number
  (`+CC...`), or a local number together with `country` (see the table below).
  Without `country`, a local number is treated as Saudi. Spaces, dashes,
  brackets and dots are allowed. The number is stored in the form `+CC...`
  (E.164), for example `+966512345678`.
- SMS is **not live yet**. See [Not live yet](#not-live-yet).

These GCC countries are accepted. `country` is one of `sa`, `ae`, `kw`, `qa`,
`bh`, `om`:

| `country` | Country | National number | Stored form |
| --- | --- | --- | --- |
| `sa` (default) | Saudi Arabia | `5` + 8 digits (`512345678` or `0512345678`) | `+9665XXXXXXXX` |
| `ae` | United Arab Emirates | `5` + 8 digits (`512345678` or `0512345678`) | `+9715XXXXXXXX` |
| `kw` | Kuwait | `5`, `6` or `9` + 7 digits (`51234567`) | `+965XXXXXXXX` |
| `qa` | Qatar | `3`, `5`, `6` or `7` + 7 digits (`31234567`) | `+974XXXXXXXX` |
| `bh` | Bahrain | `3` + 7 digits (`31234567`) | `+973XXXXXXXX` |
| `om` | Oman | `7` or `9` + 7 digits (`71234567`) | `+968XXXXXXXX` |

A leading `0` (trunk) in a local number is dropped.

## 1. Check whether you must verify

`GET /v1/me` returns two fields on `account`:

```json
{
  "account": {
    "id": "acc_...",
    "planId": "free",
    "phoneVerified": false,
    "phoneVerificationRequired": true
  }
}
```

| Field | Meaning |
| --- | --- |
| `phoneVerified` | `true` once a phone number is verified on this account. |
| `phoneVerificationRequired` | `true` when the account must verify a phone before it can upload (a new free account that has not verified yet). |

Use `phoneVerificationRequired` to show the phone step. Do **not** block the
whole app on it: only the first upload is blocked.

## 2. Start the code

```bash
curl -X POST "$BASE/v1/me/phone/start" \
  -H "Cookie: segmnt_session=$SEGMNT_SESSION" \
  -H "Content-Type: application/json" \
  -d '{"phone":"0512345678"}'
```

For a GCC number, send the full international number, or a local number with
`country`:

```bash
curl -X POST "$BASE/v1/me/phone/start" \
  -H "Cookie: segmnt_session=$SEGMNT_SESSION" \
  -H "Content-Type: application/json" \
  -d '{"phone":"0501234567","country":"ae"}'
```

Answer (`202`):

```json
{
  "challengeId": "phc_8fK2mQ",
  "phone": "+96651****678",
  "expiresAt": "2026-10-14T10:10:00.000Z"
}
```

Notes:

- `phone` in the answer is **masked**. Keep the full number only if you need it;
  we never return it.
- The code is **6 digits** and expires in **10 minutes** (`expiresAt`).
- At most **5 codes per account per hour**. More gets `429 phone_rate_limited`.
- Each challenge is for one phone number. Starting again makes a new challenge;
  the old one still works until it is used or it expires.
- A banned account or phone number gets `403 banned`.
- A number that is not a Saudi or GCC mobile gets `400 invalid_phone`.
- A `country` that is not one of `sa`, `ae`, `kw`, `qa`, `bh`, `om` gets
  `400 invalid_country`.
- If the SMS provider cannot send the code, you get `502 sms_send_failed`.
  Nothing is stored, so try again.

## 3. Verify the code

Send the `challengeId`, the 6-digit `code`, and a `fingerprint`.

```bash
curl -X POST "$BASE/v1/me/phone/verify" \
  -H "Cookie: segmnt_session=$SEGMNT_SESSION" \
  -H "Content-Type: application/json" \
  -d '{"challengeId":"phc_8fK2mQ","code":"123456","fingerprint":"my-stable-device-id"}'
```

Answer (`200`):

```json
{ "phoneVerified": true, "phoneVerifiedAt": "2026-10-14T10:04:00.000Z" }
```

About the `fingerprint`:

- It is a **stable identifier for the device or browser** (for example a random
  id you store in the browser, or a device hash). 1 to 256 characters.
- It is stored only as a **salted hash**, linked to the account. The raw value is
  never kept.
- A blocked account, phone number or device gets `403 banned`.

Rules for the challenge:

- **One use.** A challenge verifies one phone number once.
- **10-minute expiry.** After that, you get `410 code_expired`; start again.
- **5 wrong codes** close it. After 5 `invalid_code` answers, later codes get
  `429 too_many_attempts`; start a new code.
- **Safe to retry.** Repeating a successful call with the same `challengeId`
  returns `200` again and does not verify twice.

## 4. Upload is blocked until you verify

When your app (or the dashboard) starts an upload with
`POST /v1/projects/{projectId}/uploads`, a free account that must verify gets:

| Status | `code` | Meaning |
| --- | --- | --- |
| `403` | `phone_verification_required` | A new free account has not verified a phone yet. |
| `403` | `account_banned` | The account is blocked. |

Finish [step 3](#3-verify-the-code) first. Paid plans and accounts that existed
before this feature are not gated. The ban check applies to every plan.

The phone gate is active only when an SMS provider is configured. While SMS is
not live (see [Not live yet](#not-live-yet)), an upload is **not** blocked with
`phone_verification_required`; the ban check still applies.

## Not live yet

- **SMS sending.** Sending a real SMS costs money, so no SMS provider is enabled
  until a provider is approved and its monthly cost is agreed. Until then
  `POST /v1/me/phone/start` answers `503 sms_not_configured`. When a provider is
  approved, the same calls work with no change from you.
- **The dashboard phone step.** Until it ships, the API is the source of truth:
  read `phoneVerificationRequired` from `GET /v1/me` and call the two endpoints
  yourself.
- **A device list.** You cannot list the devices of an account yet.

In the TEST environment only, and only when the test sink is on, the code also
comes back as `testCode` in the start answer so our automated tests can finish
the flow without a real SMS. `testCode` is **never** present in production.

## Errors

Errors use `application/problem+json`. Use the `code` field in your code, not
the text. The `type` field links to `https://docs.segmnt.cloud/errors/{code}`.

| Status | `code` | Where | Meaning | What to do |
| --- | --- | --- | --- | --- |
| 400 | `invalid_request` | verify | Missing or wrong `challengeId`, `code` or `fingerprint` | Fix the request. Do not retry. |
| 400 | `invalid_phone` | start | Not a Saudi or GCC mobile number | Send a number like `05XXXXXXXX` or `+9715XXXXXXXX`. |
| 400 | `invalid_country` | start | `country` is not `sa`, `ae`, `kw`, `qa`, `bh` or `om` | Send a GCC `country`, or a full international number. |
| 400 | `invalid_code` | verify | The code is wrong | Try again. After 5 wrong codes the challenge closes. |
| 401 | `unauthorized` | both | Missing, wrong or expired session | Sign in again. |
| 403 | `banned` | both | The account, phone or device is under an active ban | Contact us if you think this is a mistake. |
| 403 | `phone_verification_required` | upload | Not verified yet | Verify a phone number first. |
| 403 | `account_banned` | upload | The account is blocked | Contact us if you think this is a mistake. |
| 404 | `challenge_not_found` | verify | Unknown, already-used or another account's challenge | Start a new code. |
| 405 | `method_not_allowed` | both | Wrong method for the URL | Use `POST`. |
| 410 | `code_expired` | verify | The 10 minutes passed | Start a new code. |
| 429 | `too_many_attempts` | verify | 5 wrong codes on this challenge | Start a new code. |
| 429 | `phone_rate_limited` | start | More than 5 codes in an hour | Wait, then try again. |
| 502 | `sms_send_failed` | start | The SMS provider could not send the code | Try again in a moment. Nothing was stored. |
| 503 | `not_configured` | both | Phone verification is off in this environment | Nothing to fix on your side. |
| 503 | `sms_not_configured` | both | No SMS provider is approved yet | Nothing to fix on your side. |

## Tested in TEST

Checked against `https://test-api.segmnt.cloud` on 14 Oct 2026, after the
phone-verification release reached TEST:

- `POST /v1/me/phone/start` with no session → `401 unauthorized`.
- `POST /v1/me/phone/verify` with no session → `401 unauthorized`.
- `GET /v1/me/phone/start` → `405 method_not_allowed` (both routes are `POST`).

The full flow is proved by our API tests in `api/src/phone.test.ts`: Saudi and
GCC E.164 normalization, an unknown `country` (`400 invalid_country`), start and
verify, the rate limit, the 10-minute expiry, the 5 wrong codes, the idempotent
retry, the upload gate, the ban checks, and a failed SMS send (`502
sms_send_failed`, with nothing stored). The signed-in path needs a real session,
so it is not re-run by hand here.
