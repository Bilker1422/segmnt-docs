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

## Protected and public videos

> **Status: Coming soon.** Today every video is **protected**. The per-video
> **public** access mode is designed but not released yet, so the setting below
> does not work on a live account. See [Feature status](/status).

Every video is **protected** today. The video plays only with a signed,
video-specific token on your verified domains. Nothing changes for your existing
videos.

When the public access mode ships, a video will also be able to be **public**: an
opt-in you turn on per video, in the video page of the dashboard ("Playback
access") or with
`PATCH /v1/projects/{projectId}/videos/{videoId}` and
`{"accessMode":"public"}`. A public video will play **with no token**, on any site
that embeds it — not only your verified domains.

A public video will still go through us: it is served from our CDN, never from a
raw storage URL, and it keeps the plan-limit checks.

**Risk:** once a video is public, anyone who has the link can share and embed it.
Make a video public only when that is what you want. You can switch it back to
**Protected** at any time.

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

## Part 3: Viewer network binding (optional)

> **Status: Coming soon.** Network binding is built; we have not offered it
> publicly yet. This section explains how it will work. See
> [Feature status](/status).

By default a playback link works from **any** network, so anyone on your verified
domain who has the link can watch it until the token expires. If that is too
loose, you will be able to bind a token to the **viewer's network**.

- You will turn it on per project with `PATCH /v1/projects/{projectId}` and
  `{"ipBinding": true}`. It is **off by default**, and it will be available on
  every plan — it does not change prices or plan limits.
- When it is on, your server will send the **viewer's** IP address as `viewerIp`
  when it creates a token. Our API sees **your server's** IP, not the viewer's,
  so your server is the one that has to know the viewer's address.
- The token will then hold the viewer's **network prefix**, not the exact
  address: IPv4 `/24`, IPv6 `/64`. A request from another network will get
  `403 ip_mismatch`, on the media path and on the embed page.
- The setting applies to tokens you create **after** you change it. A token you
  created earlier keeps working (or stays bound) until it expires. Use a short
  `expiresInSeconds` when you turn binding on.

Turn it on (session cookie required):

```bash
curl -X PATCH "https://api.segmnt.cloud/v1/projects/$PROJECT_ID" \
  -b "segmnt_session=<your session cookie>" \
  -H "Content-Type: application/json" \
  -d '{"ipBinding": true}'
```

Every project will carry `ipBinding` in its answer, so you can read it back. The
switch is documented with the other project settings in
[Accounts, projects, API keys and domains](/accounts-and-projects#playback-network-binding-optional).

You will then create a token with the viewer's IP — see
[Bind a token to the viewer's network](/playback-tokens#bind-a-token-to-the-viewers-network-optional).
On a bound project a token will **never** be minted without it: a missing or
invalid `viewerIp` will be `400 invalid_request`. Do **not** send your server's
own IP; that would bind every token to your server's network and normal viewers
would fail.

### What viewers will see

A viewer on another network (for example, someone who copied the link) will get
`403` and the player will show a short message. The answer will carry
`X-Segmnt-Error: ip_mismatch`, so your own player can tell this apart from an
expired token. Ask the viewer to open the page from their own network, or have
your server issue a fresh token for that network.

### What this does not stop

Network binding is **not DRM**, and it is not a per-person lock:

- Two viewers on the **same network** (`/24` for IPv4, `/64` for IPv6) can still
  share a link. One mobile-carrier NAT can put many viewers on one prefix.
- It does not stop **screen recording**.
- It names a **network**, not a person. It cannot tell two viewers on one
  network apart.

Use it together with the domain lock to make a copied link fail on a **different
network**. For a hard per-person limit, keep the token short and put playback
behind your own sign-in.

## What the CDN checks on every request

Each playback request goes through the same gate, in this order:

1. **Token present?** Missing → `403 token_missing`.
2. **Signature valid, and for this video?** No → `403 token_invalid`. The token
   names one video, so a token for another video never plays this one.
3. **Not expired?** Expired → `403 token_expired`.
4. **Network (only when the project turns on IP binding).** *(Coming soon — see
   [Feature status](/status).)* The request will then have to come from the
   viewer's network: IPv4 `/24` or IPv6 `/64`. Another network →
   `403 ip_mismatch`. This is off by default.
5. **Domain.** The request's `Origin` header (or the origin of its `Referer`)
   must be exactly `https://` + the token's domain. Missing → `403 origin_missing`;
   different host → `403 domain_mismatch`. (Our own CDN host is also accepted,
   for the player's own requests.)
6. **Not removed, and inside the plan's quality.** Removed → `403 video_unavailable`;
   over the plan quality → `403 plan_limit_reached`.
7. **Serve.** We add the token to every link inside the playlist, so the player
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
  own login on top, if you need that. Viewer network binding (Coming soon)
  narrows a token to one **network**, not one person.
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

## Tested against the current release

Checked on 15 Oct 2026:

- `GET https://cdn.segmnt.cloud/e/{videoId}` with no token → `403` with
  `X-Segmnt-Error: token_missing`.
- `GET https://cdn.segmnt.cloud/v/{videoId}/master.m3u8` with no token →
  `403`.
- The token check, the expiry, the `Origin`/`Referer` domain check, the
  `frame-ancestors` policy and the playlist rewrite are covered by our playback
  Worker tests.
- **Per-video binding:** a token made for one video does not play another. The
  token carries the video id, so the CDN answers `403 token_invalid` for a
  different video, on the media path and on the embed page.
