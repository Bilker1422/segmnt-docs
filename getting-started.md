---
title: Set up your first project
description: Sign in with SSO, create a project, make an API key, verify a domain, and upload your first video.
icon: "rocket"
---

This page sets up the account behind every project: sign-in, a project, an API
key and a verified domain. When it is done, go to the
[Quickstart](/quickstart) to upload a video, create a playback token, embed it
and see the `403` off your domain.

Sign-in is **SSO only** (Google or GitHub). **We never store a password.**

> العربية: سجّل الدخول عبر SSO، وأنشئ مشروعاً، وأنشئ مفتاح API، ووثّق نطاقاً،
> وارفع أول فيديو لك.

## Before you start

- Dashboard: https://app.segmnt.cloud
- API: https://api.segmnt.cloud
- IDs: accounts `acc_`, users `usr_`, projects `prj_`, API keys `key_` (the
  secret is `sk_`), domains `dom_`, videos `vid_`.
- Your free account is created on the first sign-in.

## Step 1: Sign up with SSO

1. Open the dashboard: https://app.segmnt.cloud.
2. Pick **Google** or **GitHub**.
3. The provider signs you in. We create your free account on this first sign-in
   and start a 30-day session.

No password, no card. The full detail, including the session cookie, is in
[Accounts, projects, API keys and domains](/accounts-and-projects).

## Step 2: Create your first project

A project is one client, one site or one course. An agency can keep each client
separate. A free account has **1** project.

In the dashboard: **Projects** → **New project**, then give it a name.

Or with the API (session cookie):

```bash
curl -X POST https://api.segmnt.cloud/v1/projects \
  -H "Cookie: segmnt_session=$SEGMNT_SESSION" \
  -H "Content-Type: application/json" \
  -d '{"name":"Client A courses"}'
```

The answer (`201`) carries the project `id` (`prj_...`). Keep it; the next steps
use it.

## Step 3: Create an API key

An API key is how your **server** calls the API. It belongs to one project.

1. Open the project → **API keys** → **Create key**.
2. Name it, and copy the secret (`sk_...`). We show it **once**.

Send it from your server, never from a browser:

```
Authorization: Bearer sk_...
```

You can make a **playback-only** key for a web server that only needs to create
playback tokens. It cannot upload, delete, read account data or spend money. See
[Create playback tokens on your server](/playback-tokens).

## Step 4: Verify a domain

Playback is locked to a domain, so add the host that will show the videos.

1. Open the project → **Domains** → **Add domain**.
2. Enter the exact host, for example `courses.example.com`. A subdomain is its
   own domain: add `www.courses.example.com` separately if you use it.
3. Copy the TXT record we show you, and add it at your DNS provider.
4. Click **Verify**. The status becomes **Verified** when the record is found.

A free account can verify **1** domain. The full detail is in
[Accounts, projects, API keys and domains](/accounts-and-projects).

## Step 5: Upload your first video

Uploading needs the API key from Step 3 and the verified domain from Step 4.

- The fastest path is the [Quickstart](/quickstart).
- The full upload flow, with Node.js, Python and PHP/Laravel examples, is in
  [Direct upload](/direct-upload).

Encoding runs on our side after the upload ends, so the video is `ready` a
little later. Poll the video until `status` is `ready`:

```bash
curl "$BASE/v1/projects/$PROJECT_ID/videos/$VIDEO_ID" \
  -H "Authorization: Bearer $SEGMNT_KEY"
```

The [Quickstart](/quickstart) then creates a playback token, embeds the video,
and shows the `403` off your domain.

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

- A free account is enough to build and test one site or product. It keeps our
  logo on the player and plays up to **720p**.
- Counters reset at the start of each UTC month. Stored hours do not reset; they
  fall when you delete a video.
- Hours count by the source quality, so a 4K file uses more of the limit than a
  720p file.

See [Usage and plan limits](/usage-and-limits) for the full rules.

## Where to go next

- [Quickstart](/quickstart) — upload, token, embed, `403`.
- [Direct upload](/direct-upload) — the four upload calls.
- [Manage videos](/manage-videos) — rename or delete a video, and free its stored
  hours.
- [Accounts, projects, API keys and domains](/accounts-and-projects) — the setup
  behind every project.
- [Errors](/errors) — the codes, including `403` and `409`.
- [Common questions](/faq) — short answers to the questions developers ask most.
- [Feature status](/status) — what is available, beta and coming soon.

## Tested against the current release

The routes on this page are the released API:

- `POST /v1/projects` and `GET /v1/projects/{projectId}/videos/{videoId}` are in
  the API contract and are covered by our API tests.
- The signed playback path (token issue → CDN check → playlist rewrite) and the
  iframe embed page are proved end to end by our playback tests.
