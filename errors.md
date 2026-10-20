---
title: Errors
description: The error shape, every error code, the causes of 403 and 409, and the rate limits.
icon: "triangle-alert"
---

Every API error is a problem document (`application/problem+json`, RFC 9457):

```json
{
  "type": "https://docs.segmnt.cloud/errors/plan_limit_reached",
  "title": "Plan limit reached.",
  "status": 403,
  "code": "plan_limit_reached",
  "detail": "A free account can verify 1 domain."
}
```

Read the **`code`** field in your code. The `title` and `detail` are for humans
and can change; the `code` is stable. The `type` is a stable URI for the code.

| Field | Meaning |
| --- | --- |
| `type` | A URI that names the error code. |
| `title` | A short human sentence. |
| `status` | The HTTP status code. |
| `code` | The stable machine value. **Branch on this.** |
| `detail` | A longer human sentence. |

## Rate limits

Every `/v1/**` and `/auth/**` request is limited by client IP.

| Area | Limit |
| --- | --- |
| `/v1/**` | 1,200 requests per minute, per client IP |
| `/auth/**` | 60 requests per minute, per client IP |

`GET /health` and `OPTIONS` preflight requests are never counted. Over the limit
you get `429` with code `rate_limited` and a `Retry-After` header, in seconds.
Wait that many seconds, then retry.

## All codes

### 400 — the request is wrong

| `code` | Cause |
| --- | --- |
| `invalid_request` | A missing or wrong field, a bad cursor, a bad date range, or a body that is not JSON. |
| `invalid_parts` | The upload `parts` list does not match what was uploaded. |
| `invalid_state` | The sign-in flow expired or was changed. Start sign-in again. |
| `auth_denied` | The user cancelled sign-in at the provider. |
| `billing_mismatch` | A payment does not match the order (amount, currency, plan or account). |

### 401 — you are not authenticated

| `code` | Cause |
| --- | --- |
| `unauthorized` | Missing, wrong, expired or revoked session or API key. |

### 403 — you are authenticated, but not allowed

This is the most useful code to handle. Read `code` to know which 403 you have.

| `code` | Where | Cause | What to do |
| --- | --- | --- | --- |
| `plan_limit_reached` | API | You are over a plan limit: stored hours, new uploads, watched minutes, projects, domains, file size, or a quality above your plan's maximum. | Delete something, wait for the period to end, or change plan. |
| `domain_not_verified` | API | You asked for a playback token for a domain that is not verified on the project. | Verify the domain first. |
| `insufficient_scope` | API | A playback-only API key called a route it may not use. | Use a full API key, from your server. |
| `banned` | API | The account or device is under an active ban. | Contact us if you think this is a mistake. |
| `account_banned` | API | The account is blocked. | Contact us if you think this is a mistake. |
| `token_missing` | CDN | No `token` query parameter on a protected video. | Send a token. |
| `token_invalid` | CDN | The token is fake, or it is for another video. | Send a token made for this video. |
| `token_expired` | CDN | The token is past its `expiresAt`. | Ask your server for a new token. |
| `ip_mismatch` | CDN | Coming soon: the project has IP binding on and this request is from another network (or the viewer IP cannot be checked). | Play from the network the token was made for, or ask your server for a token from this network. |
| `origin_missing` | CDN | The request has no `Origin` and no `Referer` header. | Play from a browser page, not a bare request. |
| `domain_mismatch` | CDN | The request's origin is not the token's domain. | Play from your verified domain. |
| `video_unavailable` | CDN | The video is held or deleted. | Stop playback. Wait for `video.ready`. |
| `plan_limit_reached` | CDN | The account is over its watched-minutes limit, the request is for a quality above the plan, or the request comes from your own site on a plan that plays in our player (Free and Starter). | Change plan, play a lower quality, or use the embed code. |

The CDN also sends the same value in the `X-Segmnt-Error` header, so a player can
read the cause without parsing the body. See
[How playback protection works](/playback-protection).

`ip_mismatch` applies when a project turns on viewer network binding, which is
**Coming soon**. See [Feature status](/status).

### 404 — not found

| `code` | Cause |
| --- | --- |
| `not_found` | A wrong id, an id in another account, or a video that was deleted. A route that is not deployed also answers `404`. |

### 409 — the resource is in the wrong state

A `409` means the request is valid, but the resource is not ready for it.

| `code` | Where | Cause | What to do |
| --- | --- | --- | --- |
| `project_not_empty` | Delete project | The project still has videos. | Delete the videos first. |
| `hostname_taken` | Add domain | The host is already added. | Use another host, or claim it with its owner. |
| `video_not_ready` | Playback token | The video is not `ready`, or it is held. | Wait for `video.ready`. Do not retry in a loop. |
| `upload_closed` | Upload complete / cancel | The upload was cancelled, expired, or already completed. | Start a new upload. |

### 422 — the request conflicts with an earlier one

| `code` | Cause |
| --- | --- |
| `invalid_request` | The same `Idempotency-Key` was used with a different body. Key is scoped to the account and the route. |

### 429 — too many requests

| `code` | Cause |
| --- | --- |
| `rate_limited` | Over the per-minute limit. Wait `Retry-After` seconds. |

### 500, 502, 503 — our side

| Status | `code` | Cause | What to do |
| --- | --- | --- | --- |
| 500 | `internal_error` | We could not complete the request. | Try again. |
| 502 | `provider_error` | The sign-in provider rejected the request. | Try again. |
| 502 | `billing_provider_error` | The payment provider did not answer. | Try again. A webhook is retried by the provider. |
| 503 | `not_configured` | The feature is off in this environment. | Nothing to fix on your side. |
| 503 | `sms_not_configured` | The old phone endpoints have no SMS service behind them. Phone verification is not planned. | Nothing to build. See [Phone verification](/phone-verification). |
| 503 | `billing_paused` | Paid plans are not on sale yet. | Join the paid-plan whitelist. Nothing was charged. |
| 503 | `billing_unavailable` | Online payment is not configured. | Ask us for a manual invoice. |

## A retry rule of thumb

- **Retry** `429` (after `Retry-After`) and `5xx`.
- **Do not retry** `400`, `401`, `403`, `404`, `409`, `410` or `422` as-is. Fix
  the request, or wait for the state to change.

## Where to go next

- [Create playback tokens on your server](/playback-tokens) — the `409 video_not_ready` path.
- [Direct upload](/direct-upload) — the `409 upload_closed` path.
- [Feature status](/status) — what is available, beta and coming soon.
