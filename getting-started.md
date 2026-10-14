---
title: Getting started
description: Sign up with SSO, create your first project, upload your first video, get a signed playback token, and see the free-tier limits.
icon: "rocket"
---

Set up your first video in about 5 minutes. This is the short path: each step
links to the full guide. Encoding runs on our side, so the video is `ready` a
little after the upload ends.

> العربية: أنشئ أول فيديو لك في نحو 5 دقائق: سجّل الدخول عبر SSO، وأنشئ مشروعاً،
> وارفع أول فيديو، واحصل على رمز تشغيل موقّع، واعرف حدود الخطة المجانية.

Everything on this page runs on **TEST**.

## Before you start

- Dashboard (TEST): `https://test-app.segmnt.cloud`
- API (TEST): `https://test-api.segmnt.cloud`
- Sign-in is **SSO only** (Google or GitHub). **We never store a password.**
- Your free account is created on the first sign-in.
- IDs: accounts `acc_`, projects `prj_`, API key ids `key_` (the key secret is
  `sk_`), domains `dom_`, videos `vid_`.

## Step 1: Sign up with SSO

1. Open the dashboard: `https://test-app.segmnt.cloud`.
2. Pick **Google** or **GitHub**.
3. The provider signs you in. We create your free account on this first sign-in
   and start a 30-day session.

No password, no card. Full detail:
[Accounts, projects, API keys and domains](/accounts-and-projects).

## Step 2: Create your first project

A project is one client, one site or one course. An agency can keep each client
separate. A free account has **1** project.

In the dashboard: **Projects** → **New project**, then give it a name.

Or with the API (session cookie):

```bash
curl -X POST https://test-api.segmnt.cloud/v1/projects \
  -H "Cookie: segmnt_session=$SEGMNT_SESSION" \
  -H "Content-Type: application/json" \
  -d '{"name":"Client A courses"}'
```

The answer (`201`) carries the project `id` (`prj_...`). Keep it; the next steps
use it.

## Step 3: Upload your first video

Uploading needs three things ready:

1. **An API key** for the project (`sk_...`). Create it in the dashboard, in the
   project's **API keys** tab. Send it from your **server**, never from a
   browser.
2. **A verified domain.** Playback is locked to a domain, so add the host that
   will show the videos (for example `courses.example.com`) and create the TXT
   record we give you. A free account can verify **1** domain.
3. **The upload calls.** Create the upload, `PUT` the parts, then complete. See
   [Direct upload](/direct-upload) for the full flow and for tested examples in
   Node.js, Python and PHP/Laravel.

Then poll the video until `status` is `ready`:

```bash
curl "$BASE/v1/projects/$PROJECT_ID/videos/$VIDEO_ID" \
  -H "Authorization: Bearer $SEGMNT_KEY"
```

Notes for a new free account:

- The **first videos are held** for an automatic check and a human review. They
  do not play until they are released. See
  [Moderation and held videos](/moderation).
- When SMS is live, a new free account verifies a phone number before its first
  upload. This is not part of the current onboarding version.

## Step 4: Get a playback token

When the video is `ready`, ask for a **playback token** for your verified
domain. Call this from your **server** each time you show the video:

```bash
curl -X POST "$BASE/v1/projects/$PROJECT_ID/videos/$VIDEO_ID/playback-tokens" \
  -H "Authorization: Bearer $SEGMNT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain":"courses.example.com"}'
```

The answer has `token`, `expiresAt`, `thumbnailUrl` and, on plans with
white-label, `playlistUrl`:

```json
{
  "token": "v1....",
  "expiresAt": "2026-10-14T17:00:00.000Z",
  "thumbnailUrl": "https://test-cdn.segmnt.cloud/v/vid_9xR4tL/thumb.jpg"
}
```

Notes:

- Playback is **signed and domain-locked**. The CDN compares the `Origin` header
  (or the origin of `Referer`) with the token's domain, so a copied link does not
  work on another site.
- The CDN checks the token on every request and adds it to every link inside the
  playlist, so a player only needs the master playlist URL.
- The token expires at `expiresAt` (4 hours by default). Ask for a new token for
  each viewing session.

> **The embed code is not here yet.** The player's iframe and JavaScript embed
> page, and the dashboard's copy-embed control, are **not released**: on TEST the
> embed URL still answers `404`. When the embed page ships, this step will show
> the embed code. Until then, this step ends at the playback token.

## Step 5: Know the free-tier limits

| Limit | Free |
| --- | --- |
| Stored hours | 2 h |
| New uploads | 2 h per month |
| Watched minutes | 2,000 per month |
| Projects | 1 |
| Verified domains | 1 |
| Highest quality | 720p |
| Our logo on the player | Shown |

- A free account is enough to build and test one course. It keeps our logo on
  the player and plays up to **720p**.
- Counters reset at the start of each UTC month. Stored hours do not reset; they
  fall when you delete a video.
- Hours count by the source quality, so a 4K file uses more of the limit than a
  720p file.

## Where to go next

- [Direct upload](/direct-upload) — the 4 upload calls, with code examples.
- [Accounts, projects, API keys and domains](/accounts-and-projects) — the setup
  behind every project.
- [Moderation and held videos](/moderation) — why a first video waits.
- [Reports and video review](/reports-and-review) — what a hold or a remove does
  to playback.

## Tested in TEST

The routes on this page are the released TEST API:

- `POST /v1/projects` and
  `POST /v1/projects/{projectId}/videos/{videoId}/playback-tokens` are in the API
  contract (`api/openapi.yaml`) and are covered by our API tests.
- The signed playback path (token issue → CDN check → playlist rewrite on
  `/v/{videoId}/...`) is proved end to end against TEST by
  `a7-playback-e2e.yml`.
- The iframe/JavaScript embed page (`/e/{videoId}` and `segmnt-player.js` on the
  CDN host) is **not deployed yet**. It ships with the player release, so the
  embed code is not documented here yet.
