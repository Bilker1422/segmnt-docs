---
title: Integration guide: move your videos and go live
description: One path from zero to a live integration. Create a project, pick the right API key, verify your domain, move your video library, connect webhooks, and answer the common setup questions.
icon: "route"
---

This guide is for a developer at an agency, an edtech company or a course
platform. It takes you from an empty project to a live integration: your videos
are encoded, they play only on your own domain, and your server knows when each
video is ready.

Read it in order. The last section is a seven-day checklist.

> **Status words.** Each feature here is marked **Available**, **Beta** or
> **Coming soon**. See [Feature status](/status) for what each word means. Do not
> build on a **Coming soon** feature; plan around it.

## 1. Set up your account

Sign in with Google or GitHub, create a project, and create an API key. The steps
are in [Accounts, projects, API keys and domains](/accounts-and-projects). You
can also follow the first steps in [Getting started](/getting-started).

Keep the API key on your server. Never put it in a web page, a browser or a
mobile app.

## 2. Pick the right API key

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

## 3. Verify your domain

Playback is locked to a domain. A playback token works only on the domain it was
made for. Add the host that will show the videos, then prove you control it.

1. Open the project, then **Domains**, then **Add domain**.
2. Enter the exact host, for example `courses.example.com`. A subdomain is its
   own domain. Add `www.courses.example.com` separately if you use it.
3. Add the DNS record we show you, then click **Verify**. The status becomes
   **Verified** when the record is found.

The free plan lets you verify **1** domain.

**`localhost` cannot be verified.** A domain must be a public host name with a
dot, such as `staging.example.com`. `localhost`, an IP address and a name with no
dot are refused. To test on your own machine, use a staging host you control,
for example `staging.courses.example.com`, and point it at your test page. A
copied playback link then returns `403` on any other host, which is what you want
to show your customer.

## 4. Move your video library

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
4. Wait for the `video.ready` webhook for that video (section 5), not a timer.
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

## 5. Connect webhooks

A webhook is a signed message that we send to your HTTPS endpoint when a video
changes. Use it instead of polling.

**Available:** `video.ready`, `video.failed`, `video.held` and `video.removed`.
The dashboard can add an endpoint, show its delivery log and send a test event.

To connect one:

1. Create an endpoint with a public `https://` address. We refuse `localhost`
   and private network addresses.
2. Copy the signing secret. We show it **once**. Store it like an API key.
3. On each delivery, check the `Segmnt-Signature` header against the raw body
   before you read the JSON. The full steps and a working Node.js example are in
   [Webhooks](/webhooks#verify-the-signature).
4. Answer with any `2xx` status within 10 seconds, after you have stored the
   event. Dedupe on the event `id`, because a retry sends the same event again.

If you lose the signing secret, rotate it. Rotation is available through the API;
see [Webhooks](/webhooks#rotate-the-signing-secret).

## 6. Answers to the common setup questions

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
section 2 has the details. A browser upload token is **Available**: your server
mints it, and the browser uses it for one upload.

### Can I verify `localhost` or a staging domain?

Not `localhost`. Verify a public staging host, such as `staging.example.com`,
and test on that host. Section 3 has the steps.

## 7. Your first seven days

Work through this list in order. Each step has a check you can see yourself.

- **Day 1: Create a project and an API key.** Pick the scope from section 2.
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
- [Playback tokens](/playback-tokens): how a token is made and checked.
- [Manage videos](/manage-videos): rename a video or delete it.
- [Errors](/errors): every error code and what to do about it.
- [Feature status](/status): what is Available, Beta and Coming soon.
