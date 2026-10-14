---
title: How playback protection works
description: Playback tokens and verified domains, and why a copied link does not play on another site.
icon: "shield"
---

Every segmnt playback link is **signed** and **locked to one verified domain**.
This page explains the two parts and what happens when someone copies a link.

## The short version

- A video plays only with a **playback token**.
- Your **server** creates the token with your **API key**.
- The token names **one video**, **one verified domain** and an **expiry**.
- The CDN checks the token and the domain on **every** request.
- A copied link does not play, because the other site's domain does not match
  the token.

## Part 1: Playback tokens

A playback token is a string that starts with `v1.`. It is signed with
**HMAC-SHA256**, so we can check it is real, and it holds:

| Inside the token | Meaning |
| --- | --- |
| Video id | This token plays this one video only. |
| Project id | For counting watched time. |
| Domain | The **one** host it plays on, for example `courses.example.com`. |
| Expiry | A Unix time. After it, the token stops working. |
| Max quality | The highest quality the plan allows. |

The signature is the important part. Nobody can change the token's domain or
expiry without the secret key, so a viewer cannot "unlock" a video or move it to
another site.

Default life is **4 hours**. You choose it with `expiresInSeconds` (60 to
86400). A shorter life is safer; make it at least as long as the page's cache.

## Part 2: Verified domains

A domain tells us which sites may play a project's videos. Add it in the
dashboard or with `POST /v1/projects/{projectId}/domains`, then prove you own it
with a **TXT record**. After that its status is `verified`.

- We match the **exact** host. `courses.example.com` and `www.courses.example.com`
  are two different domains. Add both if you use both.
- The API refuses a token for a domain that is not verified.
- A **free** account can verify **1** domain. See
  [Usage and plan limits](/usage-and-limits).
- Remove a domain and we stop issuing tokens for it. Tokens already issued keep
  working until they expire.

Setup details are in
[Accounts, projects, API keys and domains](/accounts-and-projects).

## What the CDN checks on every request

Each playback request goes through the same gate, in this order:

1. **Token present?** Missing → `403 token_missing`.
2. **Signature valid, and for this video?** No → `403 token_invalid`.
3. **Not expired?** Expired → `403 token_expired`.
4. **Domain.** The request's `Origin` header (or the origin of its `Referer`)
   must be exactly `https://` + the token's domain. Missing → `403 origin_missing`;
   different host → `403 domain_mismatch`. (Our own CDN host is also accepted,
   for the player's own requests.)
5. **Not removed, and inside the plan's quality.** Removed → `403 video_unavailable`;
   over the plan quality → `403 plan_limit_reached`.
6. **Serve.** We add the token to every link inside the playlist, so the player
   keeps working, and we serve the file from the cache under its **clean path** —
   the token is **not** part of the cache key.

The embed page is also locked: it sends
`Content-Security-Policy: frame-ancestors https://{domain}`, so only your domain
can frame it. A copied embed URL on another site is refused by the browser.

## Why a copied link does not play

A token is locked to one host. Copy the iframe to another site and:

- the embed page cannot be framed there (`frame-ancestors` blocks it);
- if someone copies only the video URL, the CDN sees the new site's `Origin`,
  which is not the token's domain, and answers `403 domain_mismatch`;
- an expired or missing token answers `403 token_expired` or `403 token_missing`.

So a copied link is useless on another site, and it stops working on your own
site when the token expires. Ask your server for a new token for the next
viewing session.

## What this does not stop

Protection is about **where** a video can play, not **who** can watch it.

- It does not lock a token to one viewer. Anyone on your **verified domain**
  with the link can watch it until the token expires. Use short tokens, and your
  own login on top, if you need that.
- It does not stop **screen recording**.
- It does not stop someone who can publish pages on your verified domain.

That is the normal trade-off for `hls.js` video on the open web. If you need
per-viewer limits, keep the token life short and put playback behind your own
sign-in.

## The player's logo

On the **Free** and **Starter** plans the player shows our logo. On **Pro** and
**Agency** you can white-label it: hide our logo and use your own logo and
colour. The plan and branding travel inside the signed token.

## Where to go next

- [Create playback tokens on your server](/playback-tokens)
- [Embed a video on any website](/embed-a-video)
- [Add segmnt to WordPress](/wordpress)
- [Accounts, projects, API keys and domains](/accounts-and-projects) — add and
  verify a domain.

## Tested in TEST

Checked on TEST on 15 Oct 2026:

- `GET https://test-cdn.segmnt.cloud/e/{videoId}` with no token → `403` with
  `X-Segmnt-Error: token_missing`.
- `GET https://test-cdn.segmnt.cloud/v/{videoId}/master.m3u8` with no token →
  `403`.
- The token check, the expiry, the `Origin`/`Referer` domain check, the
  `frame-ancestors` policy and the playlist rewrite are covered by the playback
  Worker tests (`playback-worker/src/token.test.ts`,
  `playback-worker/src/security.test.ts`, `playback-worker/src/embed.ts`).
