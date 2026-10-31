---
title: "From zero to a live video on your website"
description: "One path you can follow alone: sign up, verify your domain, upload a video, copy the embed code, and put it on your website. Then connect the API, webhooks and a full library move."
icon: "route"
---

This guide takes you from an empty account to a video playing on your own
website. You can follow it alone, with no support call: **Part 1** needs no
developer and no code. **Part 2** is for the developer who connects your
product through the API.

Read Part 1 in order. It ends with a video playing on your website and proof
that a copied link plays nowhere else.

> **Status words.** Each feature here is marked **Available**, **Beta** or
> **Coming soon**. See [Feature status](/status) for what each word means. Do not
> build on a **Coming soon** feature; plan around it.

---

## Part 1 — Your first video, no code

You need two things:

- A Google or GitHub account to sign in with.
- A website whose domain you control, for example `courses.example.com`.

## 1. Sign up

1. Open the dashboard: https://app.segmnt.cloud.
2. Pick **Google** or **GitHub**.

That is all. We create your free account on the first sign-in. There is no
password, no card and no phone check. Sign-up uses Google or GitHub only —
there is no email sign-up today. The detail is in
[Accounts, projects, API keys and domains](/accounts-and-projects).

## 2. Create a project

A project holds your videos, your keys and your domains. Use one project per
site, one client or one product. A free account has **1** project.

In the dashboard: **Projects** → **New project**, then give it a name.

## 3. Verify your domain

Playback is locked to your domain. A video plays only on a host you have proved
you control.

1. Open the project, then **Domains**, then **Add domain**.
2. Enter the exact host, for example `courses.example.com`. A subdomain is its
   own domain. Add `www.courses.example.com` separately if you use it.
3. Add the DNS record we show you at your DNS provider. It looks like this:

```text
Type:  TXT
Name:  _segmnt-verify.courses.example.com
Value: segmnt-verify=b41c09e2f7a3d6c1
```

4. Click **Verify**. The status becomes **Verified** when the record is found.

> **Copy the record — do not retype it.** The name starts with an underscore
> (`_`). In Arabic text a leading underscore can be shown at the wrong end of
> the name, and a retyped record then fails to verify. Copy the name and the
> value from the dashboard instead.
>
> العربية: انسخ اسم سجل DNS وقيمته من لوحة التحكم، ولا تعِد كتابتهما. يبدأ
> الاسم بشرطة سفلية `_`، وقد تظهر في موضع خاطئ داخل نص عربي.

The free plan lets you verify **1** domain. `localhost` cannot be verified —
use a public host you control, such as `staging.courses.example.com`. Full
detail is in [Accounts, projects, API keys and domains](/accounts-and-projects).

## 4. Upload your first video

In the dashboard: open the project, click **Upload**, and pick a video file.

What happens next is automatic. When the upload ends, we encode the video once
into playing levels (360p, 720p and 1080p), make the thumbnail, and lock the
video to your verified domain. The free plan plays up to **720p**.

To upload from your own code instead, follow [Direct upload](/direct-upload).

## 5. Wait until it is ready

A short video is usually ready a few minutes after the upload ends. The video
page shows the status:

| Status | Meaning |
| --- | --- |
| Uploading | The file is still going up. |
| Processing | We are encoding it. |
| Ready | It can play. Go to the next step. |
| Failed | The encode failed. See [Errors](/errors). |

You do not have to watch the page. A webhook can tell your system the moment a
video is ready — see [Connect webhooks](#connect-webhooks).

> **Note.** A review hold for the first videos of a free account is **Coming
> soon** and is off today, so a new free video is not held. When it turns on, a
> held video does not play until a person has checked it, and you get a
> `video.held` webhook. See [Moderation and held videos](/moderation).

## 6. Put the video on your website

1. Open the ready video in the dashboard.
2. Follow **Add to your website**. It shows the embed code; click
   **Copy embed code**. The copied code keeps the word `YOUR_PLAYBACK_TOKEN` —
   that is on purpose, so a real token never leaks into what you copy. (The
   first-run wizard has the same copy button, named **Copy code**, on its last
   step.)
3. Paste the code into your page, then replace `YOUR_PLAYBACK_TOKEN` with a
   real token (the next section).

The embed code looks like this:

```html
<div style="position:relative;width:100%;aspect-ratio:16 / 9;">
  <iframe src="https://cdn.segmnt.cloud/e/vid_9xR4tL?token=YOUR_PLAYBACK_TOKEN"
    style="position:absolute;inset:0;width:100%;height:100%;border:0"
    allow="autoplay; fullscreen; picture-in-picture; encrypted-media"
    allowfullscreen referrerpolicy="origin"></iframe>
</div>
```

Where to paste it:

| Where your site lives | What to do |
| --- | --- |
| WordPress | Install the **segmnt video** plugin and add `[segmnt id="vid_..."]`. The plugin handles the token for you. See [Add segmnt to WordPress](/wordpress). |
| Any page builder | Paste the iframe block into a Custom HTML block. |
| Your own app | Use the iframe block, or the JavaScript embed in [Embed a video](/embed-a-video). |

Full detail, including the JavaScript embed and the player error codes, is in
[Embed a video on any website](/embed-a-video).

## 7. Get the token (and keep it working)

A playback token is a short, signed pass for one video on one domain. It is why
a copied link plays nowhere else.

- For your first test, follow the [Quickstart](/quickstart): it creates a token
  with one call and embeds the video end to end.
- For a real page, your **server** creates a fresh token every time it builds
  the page. A token expires — by default after 4 hours, at most after 24
  hours — so a token pasted into a page once would stop playing later.

**Does this need my own server?** For a test page, no. For a real site, yes —
but it is a small job: one HTTPS call from your server mints the token for each
page view. The steps and ready-made examples (cURL, Node.js, PHP/Laravel and
Python) are in [Create playback tokens on your server](/playback-tokens). If
your site is WordPress and you cannot host code, the
[segmnt video plugin](/wordpress) mints the token for you.

## 8. Prove it is locked to your domain

Copy the video link from the embed code and open it on any other domain. The
player refuses it with `403`. That is not a bug: playback is locked to the
domain you verified in step 3. Your video cannot play on someone else's site,
and their video cannot play on yours. See
[How playback protection works](/playback-protection).

## Free-plan limits

| Limit | Free |
| --- | --- |
| Stored hours | 2 h |
| New uploads | 2 h per month |
| Watched minutes | 2,000 per month |
| Projects | 1 |
| Verified domains | 1 |
| Highest quality | 720p |
| Our logo on the player | Shown |

Counters reset at the start of each UTC month. Deleting a video frees its
stored hours. See [Usage and plan limits](/usage-and-limits) for the full rules.

## When you need more

- **Hide our logo (white-label).** It needs a paid plan. Paid plans are
  **Coming soon** — we do not promise a price or a date here. See
  [Plans and payment](/billing).
- **Your system told when a video changes** —
  [Connect webhooks](#connect-webhooks) below.
- **A whole library from another host** —
  [Move your video library](#move-your-video-library) below.

---

## Part 2 — Connect it to your product

This part is for the developer at an agency, an edtech company or a course
platform. It takes you from one embedded video to a live integration: your
videos are encoded, they play only on your own domain, and your server knows
when each video is ready.

## Pick the right API key

Each API key has one scope. Pick the scope when you create the key.

| Key or token | Status | What it can do | Where it lives |
| --- | --- | --- | --- |
| `full` key (`sk_...`) | Available | Every route of its project: uploads, videos, playback tokens, webhooks and usage. | Your server only. |
| `playback` key (`sk_...`) | Available | Only the route that creates a playback token. | Your server only. |
| Browser upload token | Available | A short-lived token so a browser can send a file without your key. It works for one upload only. | Your server creates it. Give it to the browser for that upload. |

Rules that apply to every key:

- A key works only for its own project. A key from another project gets `404`.
- A `playback` key that calls any other route is refused with the error code
  `insufficient_scope`. Use a `full` key for uploads, videos and webhooks.
- A revoked key gets `401` at once.
- An API key cannot manage projects, keys or domains. Those routes use the
  dashboard session.

For a web app, a common setup is one `full` key for your upload and admin
jobs, and one `playback` key for the server that signs playback tokens. If you
only play videos and never upload from that server, the `playback` key is enough.

## Move your video library

This is the job of moving videos that already exist, for example from your
current host or a store you own. You do not need our help to start.

**What you do:**

1. List the videos you want to move. Keep a table that maps each of your video
   IDs to the file, the title and the status.
2. For each video, download the file from your own storage. Use the source
   you already have. We do not fetch videos from other sites for you.
3. Upload each file through the upload API with your `full` key. Follow
   [Direct upload](/direct-upload): create the upload, send the parts, then
   complete it. Save the `vid_...` ID we return next to your own ID.
4. Wait for the `video.ready` webhook for that video (section below), not a timer.
5. When a viewer opens a lesson, your server creates a playback token for that
   video and your domain. The token is short-lived. Never store a token; make a
   new one each time.

**What we do:** when a video is uploaded, we encode it once to 360p, 720p and
1080p in HLS, make a thumbnail, and make review frames. The video plays when it is `ready`.

**Bulk importer: Coming soon.** A single import tool that moves a whole library
in one job is not released yet. Until it ships, a short script that calls the
upload API (steps 1 to 5 above) is the way to move a library. Read
[Status](/status) before you promise a date to your customer.

**Free plan limits:** 2 hours stored, 2,000 minutes watched each month, playback
up to 720p, and 1 verified domain. Check [Usage and limits](/usage-and-limits)
before you move a large library.

## Connect webhooks

A webhook is a signed message that we send to your HTTPS endpoint when a video
changes. Use it instead of polling.

**Available:** `video.ready`, `video.failed`, `video.held` and `video.removed`.
The dashboard can add an endpoint, show its delivery log and send a test event.

To connect one:

1. Create an endpoint with a public `https://` address. We refuse `localhost`
   and private network addresses.
2. Copy the signing secret. We show it **once**. Store it like an API key, and
   set it in **every** environment that receives webhooks. Read it when your
   server starts, and stop the server if it is missing: without it, every
   delivery fails. See [Every delivery fails](/webhooks#every-delivery-fails).
3. On each delivery, check the `Segmnt-Signature` header against the raw body
   before you read the JSON. The full steps and a working Node.js example are in
   [Webhooks](/webhooks#verify-the-signature).
4. Answer with any `2xx` status within 10 seconds, after you have stored the
   event. Dedupe on the event `id`, because a retry sends the same event again.

If you lose the signing secret, rotate it. Rotation is available through the API;
see [Webhooks](/webhooks#rotate-the-signing-secret).

## Answers to the common setup questions

### How is the signing secret encoded?

The secret is a text string that starts with `whsec_`, followed by hex
characters. Use the whole string exactly as we show it, including the `whsec_`
prefix, as the HMAC key. Do not decode it from base64 first. The signature is
the hex HMAC-SHA256 of `<t>.<raw body>`, where `t` is the timestamp in the
header.

### Are videos held when they are bulk-imported?

An import is a set of normal uploads. Each video in the import follows the same
rules as any other upload, so a bulk import does not skip a hold and does not
create a special one.

Today, the free-plan review hold is **Coming soon** and is off, so a new free
video is not held. When the hold is on, a free account's first videos wait for a
person to check them, and paid plans are not held for this reason. The automatic
frame check is also **Coming soon**. A video our team holds is held on any plan.
A held video does not play, and you get a `video.held` webhook. See
[Moderation and held videos](/moderation).

### Which API key scope do I need?

Use a `full` key for uploads, reading videos and managing webhooks. Use a
`playback` key only on a server that only creates playback tokens. The table in
[Pick the right API key](#pick-the-right-api-key) has the details. A browser
upload token is **Available**: your server mints it, and the browser uses it for
one upload.

### Can I verify `localhost` or a staging domain?

Not `localhost`. Verify a public staging host, such as `staging.example.com`,
and test on that host. The domain steps above have the detail.

## Your first seven days

Work through this list in order. Each step has a check you can see yourself.

- **Day 1: Create a project and an API key.** Pick the scope from
  [Pick the right API key](#pick-the-right-api-key).
  Check: the key is listed on the project page.
- **Day 1: Verify your domain.** Use a host you control. Check: the status is
  **Verified**.
- **Day 2: Upload one video.** Use the steps in [Direct upload](/direct-upload).
  Check: the video reaches `ready`.
- **Day 2: Play it.** Create a playback token for your domain and load the video
  on that domain. Check: it plays, and a copied link returns `403` on another
  domain. The [Quickstart](/quickstart) has the exact steps.
- **Day 3: Connect a webhook.** Add an HTTPS endpoint, copy the signing secret
  once, and send a test event. Check: the delivery log shows the event as
  delivered.
- **Day 4 to 6: Move a small batch.** Move five to ten videos with your script.
  Check: each one reaches `ready` through its `video.ready` webhook.
- **Day 7: Go live with one lesson.** Put one protected video on a real page of
  your product. Check: the page plays it, and you have a record of the video ID
  and the domain.

## Where to go next

- [Quickstart](/quickstart): one sitting from sign-up to a protected video.
- [Embed a video](/embed-a-video): the iframe and JavaScript embed codes.
- [Add segmnt to WordPress](/wordpress): the plugin path and the Custom HTML block.
- [Playback tokens](/playback-tokens): how a token is made and checked.
- [Manage videos](/manage-videos): rename a video or delete it.
- [Accounts, projects, API keys and domains](/accounts-and-projects): the setup
  behind every project.
- [Errors](/errors): every error code and what to do about it.
- [Common questions](/faq): short answers to the questions developers ask most.
- [Feature status](/status): what is Available, Beta and Coming soon.
