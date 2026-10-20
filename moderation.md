---
title: Moderation and held videos (coming soon)
description: How the planned automatic check will work, why a video will be held for review, what your app sees, and how a held video is released.
icon: "shield-check"
---

When the check is on, it runs on every video after it is ready. It looks at the
**review frames** our encoder writes (1 frame every 10 seconds). If the check
flags a video, or a free account's first videos still wait for a check, the
video is **held**: it does not play. Our team releases it from an internal
queue.

This page shows the moderation states, what your app sees when a video is held,
and how a held video is released.

> **Status: Coming soon.** The automatic frame check and the free-plan review
> hold are not on for customers yet. This page describes how moderation is
> designed to work. See [Feature status](/status).

This guide is written in simple English.

## Before you start

- API: `https://api.segmnt.cloud`
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

The free-plan review hold is **not on** today (it is **Coming soon**). When it is
on:

- A free account's **first videos** are held until a person checks them.
- A **paid** plan (**Starter**, **Pro** and **Agency**) is **never** held for
  this reason. A paid plan's first video is not held.
- The count is per account, across all of its projects, and it counts videos that
  are not deleted. After the first videos, new uploads start `none` (playable
  when `ready`).
- Until a person checks them, these videos wait in our review queue.

### Bulk imports

An upload through the API is treated the same as any other upload. There is **no
separate rule for a bulk import**: every video in the import is one upload, so the
rule above applies to each one. A bulk import does not raise the number of videos
held, and it does not skip the hold.

Phone verification is **Not planned**, so it does not hold or block the first
upload. A verified domain is the gate. See [Phone verification](/phone-verification).

A **team hold**, or a video the **automatic check** flags, is different: it can
affect any plan and starts with our team. See section 3.

## 5. How a held video is released

Our team reviews held videos on an internal page:

| Action | Result | What happens |
| --- | --- | --- |
| Approve | state `cleared` | The video is released. It plays again. |
| Hold | state `flagged` | The video stays out of playback. |

- Every Approve and Hold is **audited** as a human decision, with the reason.
- Our team's release sends the `video.ready` webhook; our team's hold sends
  `video.held`. A clean check that clears a free hold makes the video playable
  and sends no webhook.
- You **cannot** release a video yourself. If you think a hold is wrong, contact
  us.
- When we hold or remove a video we notify the account owner with the reason
  (a removal also sends `video.removed`). For a legal matter we keep the file and
  frames and do not notify you. See [Reports and video review](/reports-and-review).

## 6. Errors

Errors use `application/problem+json`. Use the `code` field in your code, not
the text.

| Status | `code` | Meaning | What to do |
| --- | --- | --- | --- |
| 409 | `video_not_ready` | The video is not `ready` yet, or it is held. `code` is always `video_not_ready`; `detail` is `The video is held for review.` for a free hold and `The video is not ready to play yet.` for a flagged or team-held video. | Wait for `video.ready`, or wait for our team to release the video. Do not retry in a loop. |
| 403 | `video_unavailable` | A playback link was used after the video was held or deleted (from the CDN) | Stop playback. Wait for `video.ready`. |
| 404 | `not_found` | Unknown video, or the key belongs to another project | Check the ids. |

## Tested against the current release

This behaviour is not on for customers yet. Our own tests cover it, and we
re-run them after every change.
