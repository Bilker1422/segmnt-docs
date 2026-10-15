---
title: Embed a video on any website
description: Copy the embed code, replace the token, paste it into your page. A short guide for non-developers.
icon: "code"
---

Put a segmnt video on your own website. It is three steps: **copy**, **replace**, **paste**. You do not need to write code.

> This is the short path. Your developer makes the token with the
> [playback token guide](/playback-tokens). [How playback protection works](/playback-protection)
> explains why the video plays only on your domain.

## Before you start

Check three things first:

1. Your video is **ready** (not "Processing", not "Held for review").
2. Your website's domain is **verified** in the project. Example: `courses.example.com`.
3. Your server can make a **playback token** for the video. See
   [Create playback tokens on your server](/playback-tokens).

Find the video id in the dashboard: open the video and look at **Details**. It
looks like `vid_9xR4tL`.

## The embed code

Copy this block into your page's HTML. Replace two things:

- `vid_9xR4tL` with your video id.
- `YOUR_PLAYBACK_TOKEN` with a token from your server.

```html
<div style="position:relative;width:100%;aspect-ratio:16 / 9;">
  <iframe src="https://cdn.segmnt.cloud/e/vid_9xR4tL?token=YOUR_PLAYBACK_TOKEN"
    title="segmnt video" loading="lazy"
    style="position:absolute;inset:0;width:100%;height:100%;border:0;"
    allow="autoplay; fullscreen; picture-in-picture; encrypted-media"
    allowfullscreen referrerpolicy="origin"></iframe>
</div>
<!-- segmnt embed · locale: en -->
```

The dashboard also builds this code for you. In the **guided setup** (the
first-run wizard, the step "Test your video and copy the embed code"), click
**Copy code**. It leaves `YOUR_PLAYBACK_TOKEN` in the code **on purpose**, so a
real token never leaks into what you copy.

> The video page's own embed panel ("Add to your website") ships with the
> dashboard redesign (YAH-703). Until then, use the guided setup, or copy the
> block above.

## Step 1: Copy the code

Copy the block above, or click **Copy code** in the dashboard's guided setup.

## Step 2: Replace the token

The `token=` value must be a **real playback token** for this video. A token:

- is made by **your server**, never in the browser;
- plays one video on one **verified domain**;
- expires, by default after **4 hours**.

Your server or your CMS replaces `YOUR_PLAYBACK_TOKEN` with a live token when it
builds the page. [Create playback tokens on your server](/playback-tokens) shows
how, with cURL, Node.js and PHP examples.

## Step 3: Paste it into your page

Paste the block where the video should appear.

- **WordPress:** install the **segmnt video** plugin (easy path), or use a
  **Custom HTML** block. See [Add segmnt to WordPress](/wordpress).
- **Another site or page builder:** paste it into an HTML block, or build it on
  your server and send the finished HTML.

## If the video does not play

The player shows a short message. The answer also carries a code:

| Code | Meaning |
| --- | --- |
| `token_missing` | The `token=` value is empty. |
| `token_invalid` | The token is fake, or it is for another video. |
| `token_expired` | The token is past its `expiresAt`. Ask your server for a new one. |
| `domain_mismatch` | The page's domain is not the token's domain. |
| `video_unavailable` | The video was removed or is held. |

## Keep the API key secret

Your **API key** creates tokens, so it is more powerful than a token. Keep it on
your server:

- never in a web page, an iframe or a Custom HTML block;
- never in a browser, a mobile app or a public repository.

A token is safe in a page: it plays one video on one domain for a short time.
The API key is not.

## Where to go next

- [Create playback tokens on your server](/playback-tokens)
- [Add segmnt to WordPress](/wordpress)
- [How playback protection works](/playback-protection)

## Tested in TEST

Checked on 15 Oct 2026:

- `GET https://cdn.segmnt.cloud/e/{videoId}` with no token → `403` with
  `X-Segmnt-Error: token_missing`.
- `GET https://cdn.segmnt.cloud/segmnt-player.js` → `200` (the player
  script is live).
- The token check, the domain check and the playlist rewrite are covered by the
  playback Worker tests (`playback-worker/src/*.test.ts`).
