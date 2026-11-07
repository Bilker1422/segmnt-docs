---
title: Errors
description: Every error code in the public API, its HTTP status, its cause, how to fix it, and whether to retry.
icon: "triangle-alert"
---

Use this page when a call fails. It lists every error code the public API and
the video CDN return.

## Read the status, then the code

1. Read the HTTP **status** to know the kind of failure.
2. Read the **`code`** to know the exact cause. Branch on `code`, not on the text.
3. Use the **Retry** column to decide what to do next.

## Error body

Most errors use the problem format (`application/problem+json`, RFC 9457):

```json
{
  "type": "https://docs.segmnt.cloud/errors/unauthorized",
  "title": "Missing or expired session.",
  "status": 401,
  "code": "unauthorized",
  "detail": "Missing or expired session."
}
```

| Field | Meaning |
| --- | --- |
| `type` | An identifier for the code. It is not a web page. Do not fetch it. |
| `title` | A short sentence for people. In the API, it is the same text as `detail`. |
| `status` | The HTTP status code. |
| `code` | The stable value for your code. It does not change. |
| `detail` | A sentence for people. The text can change between releases. |

Some responses do not use this format. These are the exceptions:

- **CDN errors** also send the code in the `X-Segmnt-Error` header. The `title`
  is a short label, and `detail` repeats it.
- **Unknown API route:** a path that does not exist returns `404` with the body
  `{"error":"not_found"}`. There is no `code` field. Match on the status.
- **CSRF refusal:** a browser request from a site that is not allowed returns
  `403` with the body `{"error":"csrf_rejected"}`. There is no `code` field.
- **Embed page errors:** the embed page (`/e/`) returns an HTML page, not a JSON
  body. It uses the same status and `code` values as the table below.

## Examples

These examples show the format of each response. The values are examples.

**1. Missing sign-in on an API call** (`401`):

```http
HTTP/2 401
content-type: application/problem+json

{"type":"https://docs.segmnt.cloud/errors/unauthorized","title":"Missing or expired session.","status":401,"code":"unauthorized","detail":"Missing or expired session."}
```

With an API key instead of a session, the text is `Missing or wrong API key.`
The code is the same.

**2. A protected video with no token** (`403`, CDN):

```http
HTTP/2 403
content-type: application/problem+json
x-segmnt-error: token_missing

{"type":"https://docs.segmnt.cloud/errors/token_missing","title":"Missing playback token","status":403,"code":"token_missing","detail":"Missing playback token"}
```

**3. A wrong HTTP method** (`405`):

```http
HTTP/2 405
content-type: application/problem+json

{"type":"https://docs.segmnt.cloud/errors/method_not_allowed","title":"Method not allowed.","status":405,"code":"method_not_allowed","detail":"Method not allowed."}
```

## Error codes

This table lists every public error code once. Codes that come from the video
CDN are marked **CDN**.

| HTTP status | `code` | Cause | Resolution | Retry? |
| --- | --- | --- | --- | --- |
| 400 | `invalid_request` | A field is missing or wrong, the body is not valid JSON, a query value is out of range, or a bad cursor is sent. The message names the field. | Fix the field the message names. | No |
| 400 | `invalid_parts` | The upload `parts` do not match the data we stored. | Send the parts you uploaded, with the `etag` from each part upload. If that fails, start a new upload. | No |
| 400 | `invalid_state` | The sign-in flow expired or changed. | Start sign-in again. | No |
| 400 | `auth_denied` | The user cancelled sign-in, or the provider sent an error. | Ask the user to sign in again. | No |
| 400 | `billing_mismatch` | **Payment provider webhook only.** A verified payment does not match the order (amount, currency, plan or account). No plan change was made. | Check the invoice in the provider dashboard. | No |
| 401 | `invalid_signature` | **Payment provider webhook only.** The webhook secret is wrong, or there is no invoice to reconcile. | Check the provider's webhook secret. | No |
| 401 | `unauthorized` | The session or API key is missing, wrong, expired or revoked. For a browser upload token: missing, wrong, forged or expired. | Send a valid key or session. For an upload token, mint a new one. | Only with a new credential |
| 403 | `plan_limit_reached` | The account is over a plan limit: stored hours, new uploads, projects, verified domains, file size, the upload token's `maxSizeBytes`, or the watched minutes. **CDN:** the quality is above the plan, or the plan plays only in our own player and the request comes from another page. | API: delete something, wait for the period to end, or change plan. CDN: play a lower quality, or use the embed code. | No |
| 403 | `forbidden` | **Coming soon:** your account role does not allow this action. | Ask an account owner or admin to do it. | No |
| 403 | `banned` | The account or device is blocked from signing up or signing in. | Contact us if you think this is a mistake. | No |
| 403 | `account_banned` | An API key was used on an account that is blocked. | Contact us if you think this is a mistake. | No |
| 403 | `insufficient_scope` | A playback-only API key called a route it cannot use. A browser upload token can only create an upload, read its own upload's status, and complete it. | Use a full API key from your server, or use the right token for the route. | No |
| 403 | `endpoint_limit_reached` | The project already has 5 webhook endpoints, the maximum. | Delete an endpoint, then add the new one. | No |
| 403 | `content_type_not_allowed` | An upload token does not allow this file type. | Use a type the token allows, or mint a token that allows it. | No |
| 403 | `origin_mismatch` | A browser upload token was used from a domain other than its `domain`. | Use the token from the domain it was made for. | No |
| 403 | `bot_challenge_failed` | **CDN:** the browser check did not pass on the embed page. | Ask the user to complete the check again. | Only after a new check |
| 403 | `token_missing` | **CDN:** the video is protected and the request has no `token`. | Send a token made by your server for this video. | No |
| 403 | `token_invalid` | **CDN:** the token is fake, changed, made for another video, or malformed. | Make a new token for this video. | No |
| 403 | `token_expired` | **CDN:** the token is past its expiry time. | Ask your server for a new token, then retry once. | Yes, with a new token |
| 403 | `ip_mismatch` | **CDN:** the token is bound to another network. This only happens when the project has viewer network binding on (`ipBinding`, off by default). | Ask your server for a token from the viewer's network. | Yes, with a new token |
| 403 | `origin_missing` | **CDN:** the request has no `Origin` and no `Referer` header. | Play from a browser page on your site. | No |
| 403 | `domain_mismatch` | **CDN:** the request's origin is not the domain in the token. | Play from the verified domain the token was made for. | No |
| 403 | `video_unavailable` | **CDN:** the video is held, removed, or its media is missing. | Wait for `video.ready`, then try again. Do not retry in a loop. | After `video.ready` |
| 404 | `not_found` | The ID is wrong, belongs to another account or project, or the video was deleted. **CDN:** the path is wrong or the media is missing. An unknown API route also returns `404` with no `code`. | Check the ID and the path. | No |
| 405 | `method_not_allowed` | The HTTP method is not allowed on this path. **CDN:** the method is not GET or HEAD. | Use the method listed in the [API reference](/api-overview). | No |
| 409 | `hostname_taken` | The host is already verified, on this project or on another one. | Use a different host, or remove the verified host first. | No |
| 409 | `project_not_empty` | You tried to delete a project that still has videos. | Delete the videos first. | No |
| 409 | `video_not_ready` | The video is not `ready` yet, or it is held for review. | Wait for `video.ready`. Do not retry in a loop. | After `video.ready` |
| 409 | `upload_closed` | The upload was cancelled, expired, or already finished. | Start a new upload. | No |
| 409 | `token_used` | The browser upload token was already used, or it expired. | Mint a new token. | No |
| 409 | `payg_not_available` | **Coming soon:** Pay-As-You-Go for uploads is not on this plan. | Change the plan, or leave the upload setting off. | No |
| 409 | `conflict` | We could not get a unique key prefix (a rare race). **Coming soon:** the member changed while you were working. | Send the request again. For a member change, reload the member and try again. | Key-prefix race: yes, once. Member change: after you reload |
| 429 | `rate_limited` | You sent too many requests in one minute. See [Rate limits](#rate-limits). | Wait the number of seconds in `Retry-After`, then retry. | Yes, after `Retry-After` |
| 429 | `encode_concurrency_limit` | Too many videos are being processed. The upload is still open. | Wait the number of seconds in `Retry-After` (60), then call complete again. The upload is not lost. | Yes, after `Retry-After` |
| 429 | `too_many_open_uploads` | **Coming soon:** this project already has 10 uploads open. | Finish or cancel one upload, then try again. An expired upload frees its slot. | Not until an upload is finished, cancelled or expired. It has no `Retry-After` |
| 500 | `internal_error` | We could not finish the request. | Try again later. | Yes, with backoff |
| 502 | `provider_error` | The sign-in provider returned an error. | Try again later. | Yes, with backoff |
| 502 | `billing_provider_error` | The payment provider did not answer. | Try again later. | Yes, with backoff |
| 503 | `storage_unavailable` | Video storage is down for a moment. On create, retry the upload. On complete, retry the complete call. | Retry the same call with backoff. | Yes, with backoff |
| 503 | `block_check_unavailable` | **CDN. Coming soon:** we could not finish the playback check. This is temporary and not a problem with your video or your plan. | Retry after the `Retry-After` seconds (5). | Yes, after `Retry-After` |
| 503 | `not_configured` | The feature is not turned on yet, such as sign-in or webhooks. | Nothing to fix on your side. Contact us if it stays. | No |
| 503 | `billing_paused` | Paid plans are not on sale yet. Nothing was charged. | Join the paid-plan list. See [Plans and payment](/billing). | No |
| 503 | `billing_unavailable` | Online payment is not set up. | Ask us for a manual invoice. | No |
| 503 | `upload_not_configured` | Upload token signing is not turned on yet. | Nothing to fix on your side. | No |
| 503 | `playback_not_configured` | Playback signing is not turned on yet. | Nothing to fix on your side. | No |
| 503 | `preview_not_configured` | Preview tokens are not turned on for the dashboard origin yet. | Nothing to fix on your side. | No |
| 503 | `demo_not_configured` | The demo video is not turned on yet. | Nothing to fix on your side. | No |

If a response has a status or `code` that is not in this table, do not guess.
Retry `5xx` with backoff, and `429` after `Retry-After`. Stop on any other `4xx`
status, and report the full response to us.

## Rate limits

Rate limits protect the API. Limits are counted per minute.

| Route | Counted by | Over the limit |
| --- | --- | --- |
| `/v1/**` | Client IP | `429 rate_limited` |
| `/auth/**` | Client IP | `429 rate_limited` |
| Playback token creation | API key | `429 rate_limited` |
| Upload token creation | API key | `429 rate_limited` |
| Browser upload create | Client IP | `429 rate_limited` |
| Browser upload complete | Client IP | `429 rate_limited` |
| Server upload create and complete, with an API key | API key | `429 rate_limited` |
| Public demo playback | Client IP | `429 rate_limited` |

`rate_limited` and `encode_concurrency_limit` send a `Retry-After` header. Its
value is a number of seconds. Wait that long, then retry. The `/health` route and
CORS preflight requests are never counted.

| Scope | Limit |
| --- | --- |
| `/v1/**` | 1,200 per minute |
| `/auth/**` | 60 per minute |
| Playback-token mint | 300 per minute per key |
| Upload-token mint | 300 per minute per key |
| Browser upload create | 60 per minute per IP |
| Browser upload complete | 60 per minute per IP |
| Server upload create and complete, with an API key | 1,000 per minute per key |
| Demo | 20 per minute per IP |

Your server is one client IP. If many of your users share one server, they can
reach the limit together. Batch requests, or spread calls over time.

## Errors that need extra care

### `storage_unavailable` (503)

Video storage is down for a short time. The response uses the problem format.
It does not include a `Retry-After` header.

- **On create:** no upload started. Call create again.
- **On complete:** the upload is not lost. Call complete again with the same
  parts.

Retry with backoff: wait longer after each try. If it still fails after several
tries, stop and try again later.

### `encode_concurrency_limit` (429)

Too many videos are processing. Your upload stays open. The response sends
`Retry-After: 60`. Wait 60 seconds, then call complete again with the same
parts. If it fails again, use backoff, the same as for `storage_unavailable`.

### `video_not_ready` (409) and `video_unavailable` (403, CDN)

These mean the video is not ready to play. Do not loop on the request. Wait for a
`video.ready` webhook, then request a token. See [Webhooks](/webhooks).

### Playback tokens

A `token_expired` or `ip_mismatch` error needs a new token, not a retry with the
same token. Create tokens on your server each time you show a video. See
[Create playback tokens on your server](/playback-tokens).

## Retry rule of thumb

- **Retry** `429` after `Retry-After`. Retry `5xx` with backoff. **Coming soon:** the one `429`
  without a header is `too_many_open_uploads`: wait until one upload ends.
- **Do not retry** `400`, `401`, `403`, `404`, `405` or `409` as they are. Fix the
  request, get a new credential or token, or wait for the state to change. The
  one exception is a key-prefix `conflict` (409): retry it once.
- **Do not retry in a loop** a `503` with a `not_configured`, `*_not_configured`,
  `billing_paused` or `billing_unavailable` code. Nothing changes until we change
  the setup.

## Where to go next

- [Create playback tokens on your server](/playback-tokens): the `video_not_ready` and token paths.
- [Direct upload](/direct-upload): the `upload_closed`, `token_used` and `storage_unavailable` paths.
- [How playback protection works](/playback-protection): the CDN `403` codes.
- [Feature status](/status): what is available, beta and coming soon.
