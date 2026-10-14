---
title: Reports and video review
description: File a viewer report, and understand what happens when our team holds or removes a video.
icon: "flag"
---

A viewer may report a video, for example for abuse. Your server files the report
to segmnt, and every report raises an alert for our team at once. We may then
hold the video or remove it. This page shows the report call and what a hold or
a remove does to playback.

> العربية: قد يُبلغ أحد الزوّار عن فيديو، مثلاً لمحتوى مسيء. ويرسل خادمك البلاغ
> إلى segmnt، وكل بلاغ يرفع تنبيهاً لفريقنا فوراً. وقد نوقف الفيديو مؤقتاً أو
> نزيله بعد ذلك. وتوضّح هذه الصفحة نداء الإبلاغ وما يفعله الإيقاف أو الإزالة
> بالتشغيل.

This guide is written in simple English so it is easy to translate to Arabic.
The Arabic version is on this site: use the language picker at the top right.

## Before you start

- API in TEST: `https://test-api.segmnt.cloud`
- `POST /v1/projects/{projectId}/videos/{videoId}/reports` accepts a **project
  API key** (`Authorization: Bearer sk_...`) — the same key you use for playback
  tokens.
- The player has **no "report" button yet**. Your own page collects the report
  and your server sends it to segmnt.
- Times are UTC.

## 1. File a viewer report

`POST /v1/projects/{projectId}/videos/{videoId}/reports`

Body:

| Field | Required | Notes |
| --- | --- | --- |
| `reason` | Yes | A short reason. 1 to 100 characters. |
| `detail` | No | More text for our team. Up to 1000 characters. |

```bash
curl -X POST "$BASE/v1/projects/$PROJECT_ID/videos/$VIDEO_ID/reports" \
  -H "Authorization: Bearer $SEGMNT_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $REPORT_ID" \
  -d '{"reason":"abuse","detail":"A viewer says the clip shows violence."}'
```

Answer (`201`), or `200` when the same `Idempotency-Key` was already used:

```json
{
  "id": "rep_7hK2mQ",
  "videoId": "vid_9xR4tL",
  "reason": "abuse"
}
```

Notes:

- The report is stored against the video and the project of the key. A key for
  another project gets `404 not_found`, the same as an unknown video.
- Every accepted report raises an alert for our team at once (section 2).
- The answer is always the stored report, and nothing about what we do next
  (section 3).
- Send an `Idempotency-Key` header (1 to 255 characters) to make a retry safe.
  The key is unique per project: a retried `POST` returns the first report with
  `200` and raises no second alert. Without the header, a report is
  at-least-once, so dedupe on your side when you can.

## 2. What our team does with a report

- Every report appears in our internal review list at once. A person reads it.
- If the report is real, we hold the video or remove it (section 3). A hold is
  reversible; a remove is not.
- Our review tools are internal. They are not part of this API and customers
  cannot call them.

## 3. When we hold or remove a video

We can change a video from our side:

| Action | Result | What happens |
| --- | --- | --- |
| Hold | status `held` | The video does not play. Playback stops within about a minute, also for playback links that were issued before the hold. |
| Unhold | status `ready` | The video plays again. |
| Remove | the video is deleted | The video stops playing within about a minute. `GET /v1/projects/{projectId}/videos/{videoId}` then answers `404`. |

We send a webhook on a hold:

| Event | When it is sent |
| --- | --- |
| `video.held` | We moved the video to `held`. It does not play. |
| `video.ready` | The video can play again: a new encode finished, or the hold ended. |

Handle `video.held` in your app: hide the player, or show a short message. You
receive these events only if your webhook endpoint subscribed to them.

A hold does not delete your file or reset your usage. To stop a video forever,
ask us to remove it.

## 4. New accounts

Every new account is checked by a person on our team. This runs in the
background: you never see it, and you do not have to do anything. If we find a
problem, we may hold or remove a video (section 3).

## 5. Errors

Errors use `application/problem+json`. Use the `code` field in your code, not
the text.

| Status | `code` | Meaning | What to do |
| --- | --- | --- | --- |
| 400 | `invalid_request` | `reason` is missing or longer than 100 characters, `detail` is longer than 1000 characters, `Idempotency-Key` is not 1 to 255 characters, or the body is not valid JSON | Fix the body. Do not retry as-is. |
| 401 | `unauthorized` | Missing, wrong or expired API key | Check the key. |
| 404 | `not_found` | Unknown video, or the key belongs to another project | Check the ids. |
| 405 | `method_not_allowed` | Wrong method for the URL | Use `POST`. |

## Tested in TEST

Checked against `https://test-api.segmnt.cloud` on 14 Oct 2026, once A12 was on
TEST:

- `POST .../reports` with no key, and with a wrong key, both answer
  `401 unauthorized`.
- `GET .../reports` answers `405 method_not_allowed`.
- The full path with a project key is proved by the A12 API tests
  (`api/src/alerts.test.ts`): `201` for a new report, `200` for a retry with the
  same `Idempotency-Key`, `400 invalid_request` for a missing `reason`, and
  `404 not_found` for a key of another project.

We re-run these checks after every change to the route.
