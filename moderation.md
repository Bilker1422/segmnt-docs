---
title: Moderation and held videos
description: How the automatic check works, why a video is held for review, what your app sees, and how a held video is released.
icon: "shield-check"
---

A moderation check runs on every video after it is ready. It looks at the
**review frames** our encoder writes (1 frame every 10 seconds). If the check
flags a video, or a free account's first videos still wait for a check, the
video is **held**: it does not play. Our team releases it from an internal
queue.

This page shows the moderation states, what your app sees when a video is held,
and how a held video is released.

> **Status.** This page is **TEST only**. See the "Tested in TEST" section below.

This guide is written in simple English.

## Before you start

- API in TEST: `https://test-api.segmnt.cloud`
- Playback uses
  `POST /v1/projects/{projectId}/videos/{videoId}/playback-tokens`.
- A held video is refused with `409` `code: video_not_ready`. The `detail` is
  `The video is held for review.` for a free hold, and
  `The video is not ready to play yet.` for a flagged or team-held video.
- Times are UTC.

## 1. What the check does

- The encoder writes 1 review frame every 10 seconds to our storage.
- A moderation job reads up to 20 frames and asks the AI provider.
- The job runs on our 15-minute cron. Our team can also run it on demand.
- A **flagged** video is held. A clean check on a video that waits for a check
  releases it.
- Every decision is **audited**: what decided (the check or a person), the
  decision, the reason and the time.

The check is safe to repeat: a video has one claim, so a second scan does not
check it twice. A check that fails is retried by a later scan.

## 2. Moderation states

Each video has one moderation state:

| State | Plays? | Meaning |
| --- | --- | --- |
| `none` | Yes, when `ready` | No check is pending. |
| `pending` | No | Held until checked. Every one of a free account's first videos starts here. |
| `cleared` | Yes | Checked and allowed. |
| `flagged` | No | The check flagged it. Held until a person decides. |

Notes:

- The state is for our team. Your API sees the **effect**: the video `status`
  becomes `held`, and a playback token is refused (section 3).
- A free account's first videos are `pending` while the video `status` may
  still be `ready`. So `status: ready` alone does not mean the video plays.
  Ask for a playback token to be sure, or wait for `video.ready`.

## 3. What your app sees when a video is held

- `POST /v1/projects/{projectId}/videos/{videoId}/playback-tokens` answers
  `409` and issues no token. The `code` is always `video_not_ready`; the
  `detail` depends on the state:

  | The video | `status` | `detail` |
  | --- | --- | --- |
  | A free hold that waits for a check | `ready` | `The video is held for review.` |
  | Flagged by the check, or held by our team | `held` | `The video is not ready to play yet.` |

  Example (a free hold):

  ```bash
  curl -X POST "$BASE/v1/projects/$PROJECT_ID/videos/$VIDEO_ID/playback-tokens" \
    -H "Authorization: Bearer $SEGMNT_KEY" \
    -H "Content-Type: application/json" \
    -d '{"domain":"courses.example.com"}'
  ```

  ```json
  {
    "type": "https://docs.segmnt.cloud/errors/video_not_ready",
    "title": "The video is held for review.",
    "status": 409,
    "code": "video_not_ready",
    "detail": "The video is held for review."
  }
  ```

- A **flagged** video, and a video our team holds, gets `status: held`, and we
  send the `video.held` webhook (if your endpoint subscribed to it).
- A playback link that was issued **before** the hold stops working within about
  a minute. The CDN answers `403` with the header
  `X-Segmnt-Error: video_unavailable`.
- `GET /v1/projects/{projectId}/videos/{videoId}` still returns the video. Its
  `status` is `held` for a check or a team hold; it can be `ready` for a free
  account's first videos that still wait for a check.

Handle a held video in your app: hide the player, or show a short message. Do
not retry the playback-token call in a loop; the state changes only when a
person acts (section 5) or a check releases the video.

## 4. Free accounts: the first videos

- A free account's **first videos** are held until checked. The default is
  **3**, and our config allows 0 to 5.
- A **paid** account's first video is not held.
- Phone verification is available but not required in this version, so it does
  not hold or block the first upload. See
  [Phone verification](/phone-verification).
- Until the AI check runs, these videos wait for a person in our review queue.

The count is per account, across all of its projects, and it counts videos that
are not deleted. After the first videos, new uploads start `none` (playable when
`ready`).

## 5. How a held video is released

Our team reviews held videos on an internal page at `/admin` on the dashboard
host, in the **Moderation** tab:

| Action | Result | What happens |
| --- | --- | --- |
| Approve | state `cleared` | The video is released. It plays again. |
| Hold | state `flagged` | The video stays out of playback. |

- Every Approve and Hold is **audited** as a human decision, with the reason.
- Our team's release sends the `video.ready` webhook; our team's hold sends
  `video.held`. A clean check that clears a free hold makes the video playable
  and sends no webhook.
- You **cannot** release a video yourself. `/v1/admin/**` is for our team only.
  If you think a hold is wrong, contact us.

## 6. The AI provider (our side)

The AI provider costs money, so choosing or using one needs **board approval**
first. Today only the **no-cost `stub` provider** can be enabled
(`MODERATION_PROVIDER=stub`). Any other value is ignored, so nothing paid can be
switched on by config alone.

Until a paid provider is approved, the pipeline still works: a free account's
first videos are held and wait for a person in the review queue, exactly as
"held until checked" requires.

Config (our side):

| Variable | Meaning | Default |
| --- | --- | --- |
| `MODERATION_PROVIDER` | The AI provider. Only `stub` is accepted today. | unset (no AI check) |
| `FREE_MODERATION_HOLD_VIDEOS` | How many of a free account's first videos are held. Clamped to 0-5. | `3` |
| `MODERATION_MAX_FRAMES` | Review frames sent to the provider per video. | `20` |
| `MODERATION_STUB_FLAG` | TEST only: `1` makes the stub flag every video. | unset |

The state lives in migration `0013_moderation.sql`.

Admin API (our team only, under the **Admin** tag in the API reference):

| Route | What it does |
| --- | --- |
| `GET /v1/admin/moderation` | The human review queue: held and awaiting-check videos, oldest first. |
| `POST /v1/admin/moderation/scan` | Run the check now (safe any time; a video is checked once). |
| `GET /v1/admin/videos?moderationState=...` | Filter the admin video list by state (`none`, `pending`, `cleared`, `flagged`). |
| `POST /v1/admin/videos/{videoId}/hold` and `.../unhold` | Hold or release a video. Each is audited as a human decision. |

The admin page `/admin` has a **Moderation** tab that lists this queue and can
Approve or Hold a video. These routes sit behind Cloudflare Access and are not
for customers.

## 7. Errors

Errors use `application/problem+json`. Use the `code` field in your code, not
the text.

| Status | `code` | Meaning | What to do |
| --- | --- | --- | --- |
| 409 | `video_not_ready` | The video is not `ready` yet, or it is held. `code` is always `video_not_ready`; `detail` is `The video is held for review.` for a free hold and `The video is not ready to play yet.` for a flagged or team-held video. | Wait for `video.ready`, or wait for our team to release the video. Do not retry in a loop. |
| 403 | `video_unavailable` | A playback link was used after the video was held or deleted (from the CDN) | Stop playback. Wait for `video.ready`. |
| 404 | `not_found` | Unknown video, or the key belongs to another project | Check the ids. |

## Tested in TEST

It is **TEST only**; no paid provider is called. TEST deploys from `main`.

The behaviour is proved by our API test suite:

- `api/src/moderation.test.ts` — only `none` and `cleared` play; the free hold
  counts the account's videos and is clamped to 0-5; the check flags and holds a
  video and audits it; a clean check releases a free hold; a second run is
  skipped; a failed check is retried; only `stub` can be enabled; the scan is a
  no-op with no provider.
- `api/src/uploads.test.ts` — a free account's first 3 videos are `pending` (and
  each hold is audited); the 4th is `none`; a paid account's first video is
  `none`.
- `api/src/playback-tokens.test.ts` — `409` for `pending` and for `flagged`,
  `201` for `cleared`. (The test asserts the status only; the `detail`
  difference in section 3 comes from the guard order in `playback-tokens.ts`.)
- `api/src/admin.test.ts` — the review queue lists `pending`/held videos, a
  release clears the hold and writes the audit row, and the scan route answers.

A live check against `https://test-api.segmnt.cloud` needs a held video and a
verified domain. The API tests above cover the behaviour; we re-run them after
every change.
