---
title: "Quickstart: play one protected video"
description: In one sitting, sign up, upload a video, wait until it is ready, create a playback token, embed the video, and prove it is locked to your domain.
icon: "play"
---

This is a tutorial. Follow it from top to bottom and you will end with a
protected video playing on your own domain, and proof that a copied link returns
`403`. It takes about 10 minutes.

You need one thing first: a website or test page whose domain you control, for
example `courses.example.com`.

> Want the reference instead of a lesson? See [Set up your first project](/getting-started)
> and [Direct upload](/direct-upload).

## Step 1: Sign in

1. Open https://app.segmnt.cloud.
2. Pick **Google** or **GitHub**.

We create your free account on this first sign-in. There is no password and no
card.

## Step 2: Create a project

A project is one site, one client or one product. A free account has one project.

In the dashboard, open **Projects** → **New project**, and give it a name.

Keep the project id (`prj_...`); the next steps use it.

## Step 3: Verify your domain

Playback is locked to a domain, so add the host that will show the video.

1. Open the project → **Domains** → **Add domain**.
2. Enter the exact host, for example `courses.example.com`. A subdomain is its
   own domain: add `www.courses.example.com` separately if you use it.
3. Copy the TXT record we show you, and add it at your DNS provider.
4. Click **Verify**. The status becomes **Verified** when the record is found.

Copy the record name and value instead of retyping them: the name starts with
`_`, and in a right-to-left interface (Arabic) the underscore can appear at the
wrong end of the line.

Full detail: [Accounts, projects, API keys and domains](/accounts-and-projects).

## Step 4: Create an API key

1. Open the project → **API keys** → **Create key**.
2. Name it, and copy the secret (`sk_...`). We show it once.

Keep this key on your **server**. Never put it in a page.

## Step 5: Upload a video

The fastest path is the dashboard: open the project, click **Upload**, and pick a
short MP4.

To upload from your own code, use the four calls in
[Direct upload](/direct-upload). They send the file straight to storage with a
pre-signed URL, so the file never passes through the API.

## Step 6: Wait until the video is ready

We encode the video after the upload ends. Poll the video until `status` is
`ready`:

```bash
export BASE="https://api.segmnt.cloud"
export SEGMNT_KEY="sk_..."          # your API key (server only)
export PROJECT_ID="prj_7kQ2mB"
export VIDEO_ID="vid_9xR4tL"

curl "$BASE/v1/projects/$PROJECT_ID/videos/$VIDEO_ID" \
  -H "Authorization: Bearer $SEGMNT_KEY"
```

Read `status` in the answer:

| `status` | Meaning |
| --- | --- |
| `uploading` | The file is still going up. |
| `processing` | We are encoding it. |
| `ready` | It can play. Go to Step 7. |
| `failed` | The encode failed. |

> **Note — the review hold is Coming soon.** When the review hold is on, a free
> account's first videos wait for review. Such a video can still report `ready`,
> but the playback-token call in Step 7 is refused with `409 video_not_ready`.
> If that happens, read [Moderation and held videos](/moderation). The hold is
> off today, so a new free video is not held.

## Step 7: Create a playback token

A **playback token** is a short, signed pass for one video on one domain. Create
it on your server:

```bash
curl -X POST "$BASE/v1/projects/$PROJECT_ID/videos/$VIDEO_ID/playback-tokens" \
  -H "Authorization: Bearer $SEGMNT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain":"courses.example.com"}'
```

The answer is `201` and holds `token`, `expiresAt` and `embedUrl`. Keep the
`token` value:

```bash
export TOKEN="v1...."               # the token from the answer above
```

A token lives for 4 hours by default. For a real page, ask for a new token each
time you show the video, from your server. See
[Create playback tokens on your server](/playback-tokens).

## Step 8: Embed the video

Put this block on the page that you verified in Step 3. Replace `vid_9xR4tL` and
the token with your own values:

```html
<div style="position:relative;width:100%;aspect-ratio:16 / 9;">
  <iframe src="https://cdn.segmnt.cloud/e/vid_9xR4tL?token=YOUR_PLAYBACK_TOKEN"
    title="segmnt video" loading="lazy"
    style="position:absolute;inset:0;width:100%;height:100%;border:0;"
    allow="autoplay; fullscreen; picture-in-picture; encrypted-media"
    allowfullscreen referrerpolicy="origin"></iframe>
</div>
```

Open your page. The video plays. That is your first protected video.

The page needs nothing in the `<head>`. To make the first frame start sooner,
you can add the two optional lines in
[Open the CDN connection early](/embed-a-video#open-the-cdn-connection-early-optional).

## Step 9: Prove it is locked to your domain

Now try to use the video somewhere else. Ask for the video with a **different**
origin — a site you did **not** verify:

```bash
curl -i "https://cdn.segmnt.cloud/v/$VIDEO_ID/master.m3u8?token=$TOKEN" \
  -H "Origin: https://someone-elses-site.example"
```

The answer is `403` with the header `X-Segmnt-Error: domain_mismatch`. Ask with no
token at all, and you get `403 token_missing`:

```bash
curl -i "https://cdn.segmnt.cloud/e/$VIDEO_ID"
```

The token is bound to your domain, so a copied link does not play anywhere else.
This is the point of the product. Full detail:
[How playback protection works](/playback-protection).

## What you built

- A project with a verified domain and an API key.
- A video that we encoded to HLS.
- A signed playback token for one video and one domain.
- An embed that plays on your domain, and `403` off it.

## Where to go next

- [Embed a video on any website](/embed-a-video) — the iframe and JavaScript embeds.
- [Create playback tokens on your server](/playback-tokens) — cURL, Node.js and PHP.
- [Add segmnt to WordPress](/wordpress) — the plugin path.
- [Direct upload](/direct-upload) — the full upload flow in three languages.
- [Errors](/errors) — what `403` and `409` mean and how to react.
- [Feature status](/status) — what is available, beta and coming soon.

> Tested with the current release. The video events, tokens, domain lock and
> embed are the released API. If a step does not work for you, tell us which
> step and which answer you got.
