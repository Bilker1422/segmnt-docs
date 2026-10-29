---
title: Usage and plan limits
description: Read your stored and watched usage, see your plan limits, and know what happens when you reach a limit.
icon: "gauge"
---

Your plan sets limits on stored hours, new uploads, watched minutes, domains,
projects and playback quality. This page shows how to read your usage and your
limits, and what happens when you reach one.

This guide is written in simple English.

## Before you start

- API: `https://api.segmnt.cloud`
- `GET /v1/billing/plan` uses your **session cookie** (the dashboard session).
- `GET /v1/projects/{projectId}/usage` accepts a **project API key**
  (`Authorization: Bearer sk_...`) or your session cookie.
- Days are UTC dates in the form `YYYY-MM-DD`.

Both answers carry numbers in **seconds** or **minutes**. Stored time is in
seconds; watched time is in minutes and is rounded down.

## 1. Usage of one project

`GET /v1/projects/{projectId}/usage` returns the stored seconds of the project
and the minutes watched in a date range. It also returns your plan, its limits,
and what is left of them for the whole account this month. A project API key is
enough for all of this — no dashboard session.

| Query | Required | Default | Notes |
| --- | --- | --- | --- |
| `from` | No | Start of the current month | Inclusive. `YYYY-MM-DD`. |
| `to` | No | Today | Inclusive. `YYYY-MM-DD`. |

```bash
curl "$BASE/v1/projects/$PROJECT_ID/usage?from=2026-10-01&to=2026-10-14" \
  -H "Authorization: Bearer $SEGMNT_KEY"
```

```json
{
  "projectId": "prj_9xR4tL",
  "from": "2026-10-01",
  "to": "2026-10-14",
  "storedSeconds": 5400,
  "watchedMinutes": 842,
  "planId": "free",
  "planStatus": "active",
  "currentPeriod": {
    "start": "2026-10-01T00:00:00.000Z",
    "end": "2026-11-01T00:00:00.000Z"
  },
  "limits": {
    "storedSeconds": 7200,
    "uploadSecondsPerPeriod": 7200,
    "watchedMinutesPerPeriod": 2000,
    "verifiedDomains": 1,
    "projects": 1,
    "maxDurationSeconds": 3600,
    "maxConcurrentEncodes": 1,
    "maxQuality": 720,
    "whiteLabel": false
  },
  "accountUsage": { "storedSeconds": 5400, "watchedMinutes": 842 },
  "remaining": { "storedSeconds": 1800, "watchedMinutes": 1158 }
}
```

Notes:

- `planId` is the plan of the account the project belongs to (`free`,
  `starter`, `pro` or `agency`). `planStatus` is one of `active`, `pilot`,
  `past_due`, `canceled` — the same values as `GET /v1/billing/plan`. While a
  pilot is inside its 30-day window, the `limits` in the answer are the pilot
  limits, not the plan's own limits.
- `currentPeriod` is the current UTC month. The `start` is inclusive and the
  `end` is exclusive. The monthly counters reset when a new period starts.
- `limits` are the limits of the account's plan. In `limits`, `null` means **no
  limit**.
- `accountUsage` is the usage of the **whole account** (every project
  together): stored seconds now, and watched minutes in the current period.
- `remaining` is what is left of each limit: the limit minus the account usage.
  It counts the **whole account** for the **current month**, no matter which
  `from`/`to` range you asked for. It is never below `0`, and it is `null` when
  that limit does not exist.
- All of this works with a project API key. You do not need the dashboard
  session that `GET /v1/billing/plan` needs (section 2).
- `storedSeconds` is the total length of the project's videos **now**, not only
  in the period. It counts videos that are encoding, ready or held for review,
  and it uses [quality weighting](#4-quality-weighting). Deleting a video lowers
  it on the next call.
- `watchedMinutes` counts only the period you asked for.
- A project API key works only for its own project. A key for another project
  gets `404 not_found`, the same as an unknown project.
- `from` after `to`, or a bad date, gets `400 invalid_request`.

The next call returns the same account totals with the plan name and all of
the plan's limits. It needs a dashboard session.

## 2. Your plan, limits and usage

`GET /v1/billing/plan` returns your plan, its status, the current period, the
limits and your usage for the account.

```bash
curl "$BASE/v1/billing/plan" \
  -H "Cookie: segmnt_session=$SEGMNT_SESSION"
```

```json
{
  "plan": { "id": "free", "name": "Free" },
  "status": "active",
  "currentPeriod": {
    "start": "2026-10-01T00:00:00.000Z",
    "end": "2026-11-01T00:00:00.000Z"
  },
  "limits": {
    "storedSeconds": 7200,
    "uploadSecondsPerPeriod": 7200,
    "watchedMinutesPerPeriod": 2000,
    "verifiedDomains": 1,
    "projects": 1,
    "maxDurationSeconds": 3600,
    "maxConcurrentEncodes": 1,
    "maxQuality": 720,
    "whiteLabel": false
  },
  "usage": { "storedSeconds": 5400, "watchedMinutes": 842, "verifiedDomains": 1 }
}
```

Notes:

- `plan.name` is `Pilot` while a pilot account is inside its 30-day window.
- `status` is one of `active`, `pilot`, `past_due`, `canceled`.
- `currentPeriod` is the current UTC month. Watched minutes reset at the start
  of each month. The `start` is inclusive and the `end` is exclusive.
- In `limits`, `null` means **no limit**.
- `maxQuality` is the highest height you may play (`720` = 720p, `1080` = 1080p,
  `2160` = 4K): Free plays up to 720p, Starter up to 1080p. Pro and Agency are
  set to allow up to 4K, but 4K playback is **Coming soon** and not reachable
  today. See [Feature status](/status). `whiteLabel` tells you if you may hide
  our logo.
- `usage` is for the whole account: stored seconds now, watched minutes in the
  current period, and verified domains.
- This call needs the session cookie. An API key cannot read it. To read your
  plan, limits and what is left of them with an API key, use
  `GET /v1/projects/{projectId}/usage` (section 1). It returns the same
  `limits` plus `remaining`: what is left of your stored time and of this
  month's watched minutes.

## 3. Limits

| Limit | Free | Pilot (30 days) | Paid |
| --- | --- | --- | --- |
| Stored hours | 2 h | 10 h | Per plan |
| New uploads | 2 h per month | 10 h in total | Per plan |
| Watched minutes | 2,000 per month | 20,000 | Per plan |
| Verified domains | 1 | 3 | Per plan |
| Projects | 1 | 3 | Per plan |
| Highest quality | 720p | 1080p | Per plan (4K is Coming soon) |
| Our logo on the player | Shown | Hidden | Per plan |

- The monthly counters (new uploads and watched minutes) reset at the start of
  each UTC month. Stored hours do not reset; they fall when you delete a video.
  A pilot is different: its new uploads are capped for the **whole** pilot, not
  per month.
- The new-upload counter counts the videos you created in the month. An upload
  that is created but never completed does **not** count (it records no length).
  A video that is completed and then deleted **does** count in the month it was
  created: deleting frees stored hours, not the new-upload counter.
- The pilot runs for 30 days and needs no credit card. When it ends, the account
  returns to its plan (usually Free).
- Paid plan numbers come from the plan catalog. Paid plans are not on sale yet;
  see [Plans, payment and the paid-plan whitelist](/billing) for the prices and
  how to join the paid-plan whitelist.

## 4. Quality weighting

Hours count by the quality of the source file, for stored time and for uploaded
time:

| Source quality | 1 hour counts as |
| --- | --- |
| 1080p or lower | 1 hour |
| 1440p | 2 hours |
| 4K (2160p) | 4 hours |

Example: a 30-minute 4K lesson counts as **2 hours** of stored time. A 30-minute
1440p lesson counts as 1 hour. This keeps the low price honest for everyone: a
4K file costs us more to store and to send.

## 5. What happens at a limit

When the account is over a limit, the API answers `403 plan_limit_reached` on
the action that would pass it. It can happen on:

| Action | Endpoint |
| --- | --- |
| Start a new upload | `POST /v1/projects/{projectId}/uploads` |
| Get a playback link | `POST /v1/projects/{projectId}/videos/{videoId}/playback-tokens` |
| Create a project | `POST /v1/projects` |
| Add a domain | `POST /v1/projects/{projectId}/domains` |

What stops depends on the plan:

- **Free, Starter and Pilot:** uploads stop at the limit, and playback stops at
  the watched-minutes limit. Starter's wall is watch time.
- **Pro and Agency:** uploads stop at the limit. Playback never stops: past the
  included minutes, extra watch time is 5 SAR per 1,000 watched minutes before
  VAT and is added to your next invoice. No "call sales" wall and no sudden stop.

For Free, Starter and Pilot playback we stop issuing **new** playback links. A
link that was already issued keeps working until it expires (4 hours by
default). This is why playback can overrun the minute limit a little before it
stops.

`GET /v1/billing/plan` returns the extra watched minutes in
`usage.overageMinutes`. This number is **always `0` for Free, Starter and
Pilot**: those plans stop playback at the watched-minutes limit, so they are
never billed for overage. The pay-as-you-go amount applies only to **Pro and
Agency**, where you can see it here before the next invoice. The overage is
billed per **started minute** past the limit, from the raw watched seconds, and
it appears on your **next invoice**. The overage record is written now; adding
it to the payment provider's invoice is a later step.

You do not lose data. [Delete videos](/manage-videos#delete-a-video) or remove
domains to go back under a limit, or upgrade to raise it. When the month (or the
pilot) ends, the counters reset.

## 6. Errors

Errors use `application/problem+json`. Use the `code` field in your code, not
the text. The `type` field links to `https://docs.segmnt.cloud/errors/{code}`.

| Status | `code` | Meaning | What to do |
| --- | --- | --- | --- |
| 400 | `invalid_request` | `from`/`to` is not a date, or `from` is after `to` | Fix the range. Do not retry. |
| 401 | `unauthorized` | Missing, wrong or expired session or key | Sign in again, or check the key. |
| 403 | `plan_limit_reached` | Over a plan limit (stored, upload, watched, domain or project) | Delete something, wait for the period to end, or upgrade. |
| 405 | `method_not_allowed` | Wrong method for the URL | Use `GET`. |
| 404 | `not_found` | Wrong project id, or the key is for another project | Check the ids. |

## Tested against the current release

Checked on 14 Oct 2026:

- `GET /v1/billing/plan` with no session → `401 unauthorized`.
- `GET /v1/projects/{projectId}/usage` with no credential → `401 unauthorized`.
- `POST` on both routes → `405 method_not_allowed`.

The plan, limits and `remaining` fields on `GET /v1/projects/{projectId}/usage`
were added in v0.19.0 (23 Oct 2026). Our API unit tests cover them, including
that `remaining` is `null` when a limit does not exist and `0` when the account
is at or past a limit.

The full path (a project key and a session, the weighted stored total, watched
minutes, the plan limits and the `403 plan_limit_reached` cases on upload,
playback link, project and domain) is proved end to end by our tests.
The route behaviour and the limit checks are also covered by our API unit tests.
