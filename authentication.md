---
title: Authentication
description: The base URL, the Authorization header, API key scopes, and how playback tokens differ from API keys. Covers token expiry and the domain lock.
icon: "lock"
---

**TL;DR:** Your server calls the API with `Authorization: Bearer sk_...`. A
browser plays a video with a playback token in the `token` query value. The
token works for one video, on one verified domain, until it expires.

## Base URL

Use this base URL for every call:

`https://api.segmnt.cloud`

Every API route starts with `/v1`. The `/auth/*` and `/health` routes do not.
Send and receive JSON over HTTPS.

## Send your API key

Send the key in the `Authorization` header. Write the word `Bearer`, one space,
and then the key:

```bash
curl -sS "https://api.segmnt.cloud/v1/projects/$PROJECT_ID/videos" \
  -H "Authorization: Bearer $SEGMNT_KEY"
```

An API key starts with `sk_`. Then come 8 letters or digits, an underscore, and
64 hexadecimal characters. The 8 characters after `sk_` are the key prefix. The
key list shows the prefix, so you can tell your keys apart.

These answers mean the key was not accepted:

| Status | Code | Cause |
| --- | --- | --- |
| `401` | `unauthorized` | The header is missing or malformed, the secret is wrong, or the key is revoked. |
| `403` | `account_banned` | The account is blocked. |
| `403` | `insufficient_scope` | A `playback` key called a route other than the playback-token route. |
| `404` | `not_found` | The key belongs to another project. A key works only for its own project. |

## API keys and playback tokens

Two kinds of credential protect a video. They do different jobs.

| | API key | Playback token |
| --- | --- | --- |
| What it is | A secret that identifies one project. It starts with `sk_`. | A signed pass for one video on one domain. It starts with `v1.`. |
| Who creates it | You, in the dashboard. | Your server, with an API key. |
| Where it goes | Your server only, in the `Authorization` header. | A browser, in the `token` query value of a CDN URL. |
| What it can do | Call the API for its project, within its scope. | Play one video on one verified domain until it expires. It cannot call the API. |
| How long it lasts | Until you revoke it. | 4 hours by default. From 60 seconds to 24 hours. |

A playback token has three parts: `v1`, a payload, and a signature. We sign the
token with HMAC-SHA256. The payload is base64url-encoded JSON, not encrypted. A
person who has the token can read its video ID, domain and expiry. That is safe,
because the token works only for that video and that domain. Changing any part
of a token makes its signature fail.

Never put an API key in a web page, a mobile app, a URL, or a Git repository.
Put a playback token in the page, because the browser must send it.

## API key scopes

Every key has one scope. The scope limits the routes the key can call.

| Scope | Routes the key can call | Use it for |
| --- | --- | --- |
| `full` | Every `/v1` route of its project. | Your back end: uploads, videos, webhooks, usage and playback tokens. |
| `playback` | Only `POST /v1/projects/{projectId}/videos/{videoId}/playback-tokens`. | A server that only shows videos, such as the WordPress plugin. |

The dashboard creates `full` keys. To create a `playback` key, call the API with
`"scope":"playback"` in the body, from a signed-in session. See
[Accounts, projects, API keys and domains](/accounts-and-projects#key-scopes-full-and-playback).

A `playback` key can create playback tokens for any video in its project. If it
leaks, it cannot upload, delete or list your videos.

## Create, rotate and revoke keys

- **Create:** Open the project in the dashboard, then **API keys**, then
  **Create key**. We show the secret once. Store it in your server's secret
  store or an environment variable.
- **Rotate:** Create a new key, deploy it to your server, and then revoke the old
  key. Do this before the old key leaks, not after.
- **Revoke:** Revoke the key in the dashboard. A revoked key gets
  `401 unauthorized` on the next call.

If you lose a key, you cannot read it again. Create a new key and revoke the old
one.

## Create playback tokens

Only your server creates playback tokens. Call this route with a `full` or a
`playback` key:

`POST /v1/projects/{projectId}/videos/{videoId}/playback-tokens`

The body has these fields:

| Field | Required | Value |
| --- | --- | --- |
| `domain` | Yes | A host name, such as `courses.example.com`. It must be a verified domain of the project. |
| `expiresInSeconds` | No | A whole number from `60` to `86400`. The default is `14400` (4 hours). |

A `201` answer holds `token`, `expiresAt`, `embedUrl` and `thumbnailUrl`. For
complete examples in cURL, Node.js and PHP, see
[Create playback tokens on your server](/playback-tokens).

## Token expiry

The CDN checks the expiry on every request. A player requests the playlist and
each segment, so the token must last longer than the video.

- After `expiresAt`, the CDN answers `403` with `token_expired`.
- You can set the expiry from 60 seconds to 24 hours.
- We have no route that cancels one token before it expires. Keep the expiry as
  short as the video allows.

## The domain lock

A playback token holds one domain: the value that you sent in `domain`. Every
request for the video must come from that domain, or from our embed page on
the CDN. On Free and Starter, only the embed page can load the video files. A
request from your own site gets `403 plan_limit_reached`.

The CDN reads the `Origin` header. If the request has no `Origin`, it reads the
origin of the `Referer` header. The origin must match the domain exactly:

- It must use `https`.
- It must have no port number.
- Its host must equal the token's domain. `courses.example.com` does not match
  `example.com`, `www.courses.example.com`, or `http://courses.example.com`.

The domain must be verified before you can create a token. To verify a domain, see
[Step 2 in the Quickstart](/quickstart#step-2-verify-your-domain).

### CDN error codes

A refused request answers `403`. The `X-Segmnt-Error` header holds the reason.

| `X-Segmnt-Error` | Cause |
| --- | --- |
| `token_missing` | The URL has no `token` value. |
| `token_invalid` | The signature is wrong, or the token is for another video. |
| `token_expired` | The token is past its `expiresAt`. |
| `origin_missing` | The request has no `Origin` and no `Referer`. |
| `domain_mismatch` | The origin does not match the token's domain. |
| `plan_limit_reached` | The account is over a plan limit, the request asks for a quality above the plan, or the request comes from your own site on Free or Starter. |
| `video_unavailable` | The video was held or deleted after the token was made. |

## Next steps

- [Quickstart: play your first signed video](/quickstart): the full flow, with
  expected output at each step.
- [Create playback tokens on your server](/playback-tokens): cURL, Node.js and PHP examples.
- [Playback protection](/playback-protection): how the token, the domain lock and
  the CDN work together.
- [Errors](/errors): every error code and the cause of each one.
