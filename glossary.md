---
title: Glossary
description: The words we use in the docs, defined in one line each, before you meet them in a command.
icon: "book-a"
---

Read this first if a term is new. Every entry is one line.

## Playback

**Playback token** — a short, signed string your server creates for one video and
one domain. The CDN checks it on every request. It starts with `v1.`.

**Pre-signed URL** — a URL that already carries its permission, so the holder can
use it for a short time without an API key. Uploads use one pre-signed URL per
file part. A pre-signed URL is for **upload**; a playback token is for
**playback**.

**Verified domain** — a host you proved you own, by adding a DNS TXT record, for
example `courses.example.com`. Playback is locked to a verified domain.

**Domain lock** — the rule that a token plays only on its one domain. A copied
link returns `403` on another domain.

**Origin** — the scheme and host a browser page was served from, for example
`https://courses.example.com`. The CDN compares it with the token's domain.

**Access mode** — a per-video setting. **Coming soon**: `protected` (needs a
token; the default today) or `public` (plays with no token; not released yet).

## Video

**HLS** — HTTP Live Streaming. A video split into small chunks with a text
playlist that lists them. Every segmnt video is encoded to HLS.

**Rendition** — one encoded quality of a video, for example 360p, 720p or 1080p.
A video has one rendition per quality in its ladder.

**Ladder** — the set of renditions we make for a video.

**Master playlist** — the `.m3u8` file that lists the renditions. A player loads
it first, then picks a rendition. In segmnt it is
`https://cdn.segmnt.cloud/v/{videoId}/master.m3u8?token=...`.

**Segment** — one short chunk of a rendition (a few seconds of video).

**Thumbnail** — the single still image from 2 seconds into the video.

**Review frame** — one still frame every 10 seconds, kept on our side for review.

**Held** — a video that does not play while our team reviews it.

**Ready** — the status of a video after encoding finishes and it can play.

## Account

**Project** — the container for one site, client or product. It holds videos,
API keys, domains and webhook endpoints. Id prefix `prj_`.

**API key** — a server-side secret for one project, for example
`sk_Abcd1234_<64 hex>`. It authorizes the API calls for that project. Keep it on
your server, never in a page.

**Session** — the cookie from SSO sign-in (`segmnt_session`). Dashboard and
account calls use it.

**Stored hours** — the total length of the videos you host. They fall when you
delete a video.

**Watched minutes** — the minutes your viewers played this month. They reset at
the start of each UTC month.

**White-label** — hiding our logo and using your own logo and colour on the
player.

**Webhook** — an HTTPS endpoint of yours that receives a small JSON event when a
video changes state, for example `video.ready`.

**Rate limit** — a cap on requests per minute. Over it, the API answers `429`
with a `Retry-After` header.

**`Idempotency-Key`** — a header you add to repeat a create call safely. The same
key with the same body returns the first result; the same key with a different
body gets `422`.

## Plans

**Free** — the default plan. 2 stored hours, 2,000 watched minutes per month, 1
project, 1 verified domain, up to 720p, and our logo on the player.

**Pilot** — a 30-day account for a trial, with no card. See
[Usage and plan limits](/usage-and-limits).

**Starter**, **Pro**, **Agency** — the paid plans, priced in SAR per month before
VAT. Each raises the limits and the highest quality; Pro and Agency add
white-label. Paid plans are **Coming soon**: checkout is paused. See
[Feature status](/status) and [Plans and payment](/billing).

**Overage** — extra watched minutes past a plan limit. Overage is **Coming
soon**: today Free, Starter and Pilot stop playback at the limit.

## Where to go next

- [Feature status](/status) — the meaning of Available, Beta and Coming soon.
- [How playback protection works](/playback-protection) — how a token and a
  domain work together.
- [Errors](/errors) — the error codes, including `403` and `409`.
