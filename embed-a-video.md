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

Free and Starter play through our embed (our player, with our logo). Pro and
Agency can use their own player.

Copy this block into your page's HTML. Replace two things:

- `vid_9xR4tL` with your video id.
- `YOUR_PLAYBACK_TOKEN` with the `token` your server gets for this video from
  [`POST .../playback-tokens`](/playback-tokens).

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

The video page (**Add to your website**) shows the same flow as two numbered
steps: **1** get a token for this video, **2** put the token in the embed code.

Your page needs **nothing in your `<head>`** for this embed to work. If you want
the first frame to appear sooner, you can add two optional lines that open the
connection to the player CDN early. See
[Open the CDN connection early](#open-the-cdn-connection-early-optional).

## The JavaScript embed (player script in your `<head>`)

The block above is the **iframe** embed. It is the easy path, and it needs
nothing in your page's `<head>`.

If your page builder wants a script instead of raw HTML, use the
**JavaScript** embed. The player file comes from the **player CDN host**, so
its `<script>` goes in your page's `<head>`:

```html
<!doctype html>
<html lang="en">
  <head>
    <script src="https://cdn.segmnt.cloud/segmnt-player.js" defer></script>
  </head>
  <body>
    <div id="segmnt-vid_9xR4tL"></div>
    <script>
      SegmntPlayer.embed(document.getElementById("segmnt-vid_9xR4tL"), {
        videoId: "vid_9xR4tL",
        baseUrl: "https://cdn.segmnt.cloud",
        token: "YOUR_PLAYBACK_TOKEN",
        locale: "en",
      });
    </script>
  </body>
</html>
```

- The `<script src>` must point at the **player CDN host**
  (`https://cdn.segmnt.cloud`), the same host the video streams from.
- Keep it in the `<head>`, with `defer`. The browser opens the connection to
  the CDN while the rest of the page loads, so the first play is faster.
- `SegmntPlayer.embed(...)` builds the same player as the iframe block, inside
  your own page.
- Replace `vid_9xR4tL` and `YOUR_PLAYBACK_TOKEN` exactly as above.

## Open the CDN connection early (optional)

Both embeds load the player and the video from the same **player CDN host**.
This step is optional: the video plays without it. If you add these two lines to
your `<head>`, the browser opens the connection to that host while the rest of
the page loads, so the first frame can start sooner:

```html
<link rel="preconnect" href="https://cdn.segmnt.cloud">
<link rel="dns-prefetch" href="https://cdn.segmnt.cloud">
```

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
  **Custom HTML** block. See [Add segmnt to WordPress](/wordpress). For Tutor
  LMS and LearnDash lessons, see the LMS section of that guide.
- **Moodle:** see [Add segmnt to Moodle](/moodle).
- **A React app:** see [Use segmnt in a React app](/react).
- **Another site or page builder:** paste it into an HTML block, or build it on
  your server and send the finished HTML.

## If the video does not play

The player shows a short message. The answer also carries a code:

| Code | Meaning |
| --- | --- |
| `token_missing` | The `token=` value is empty. |
| `token_invalid` | The token is fake, or it is for another video. |
| `token_expired` | The token is past its `expiresAt`. Ask your server for a new one. |
| `ip_mismatch` | Coming soon: the project has IP binding on and this page is not on the token's network. |
| `domain_mismatch` | The page's domain is not the token's domain. |
| `video_unavailable` | The video was removed or is held. |

`ip_mismatch` applies when the project turns on viewer network binding, which is
**Coming soon**. See [Feature status](/status).

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

## Tested against the current release

Checked on 17 Oct 2026:

- The optional `<head>` lines are the same two tags we use on our own pages, and
  our tests cover them.
- The two lines only open a connection. A page that never contacts the CDN is
  not affected, and the video plays with or without them.

Checked on 15 Oct 2026:

- `GET https://cdn.segmnt.cloud/e/{videoId}` with no token → `403` with
  `X-Segmnt-Error: token_missing`.
- `GET https://cdn.segmnt.cloud/segmnt-player.js` → `200` (the player
  script is live).
- The token check, the domain check and the playlist rewrite are covered by our
  playback Worker tests.
- The JavaScript embed loads the same `/e/{videoId}` page and
  `/segmnt-player.js` as the iframe path. Our tests cover the embed builders;
  the browser bundle's `SegmntPlayer.embed` is not covered by them yet.
