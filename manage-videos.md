---
title: Manage videos
description: Rename a video, or delete it and free its stored hours. PATCH and DELETE examples in curl, Node.js, Python and PHP/Laravel.
icon: "sliders-horizontal"
---

After a video is uploaded you can still change it: **rename** it at any time, or
**delete** it when you no longer host that lesson. Both calls use the same
project API key (`Authorization: Bearer sk_...`) as the rest of the API, or a
dashboard session. The video keeps its `vid_...` id when you rename it.

You can also do both from your dashboard's video list. This page shows the API.

## Rename a video

`PATCH /v1/projects/{projectId}/videos/{videoId}`

Rename when you want a different label on a video: fix a typo, put the lessons
in your own order, or reuse a template. The title is only a label. Renaming does
**not** touch the file, the encoding, the watched minutes or playback.

Body:

| Field | Required | Notes |
| --- | --- | --- |
| `title` | Yes | 1 to 200 characters. Leading and trailing spaces are trimmed. |

```bash
curl -X PATCH "$BASE/v1/projects/$PROJECT_ID/videos/$VIDEO_ID" \
  -H "Authorization: Bearer $SEGMNT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title":"Lesson 2 - Advanced"}'
```

Answer (`200`): the full `Video`, with the new `title`.

```json
{
  "id": "vid_9xR4tL",
  "projectId": "prj_7kQ2mB",
  "title": "Lesson 2 - Advanced",
  "status": "ready",
  "...": "..."
}
```

Notes:

- A missing, empty, whitespace-only, too-long or non-string `title` is
  `400 invalid_request`. Nothing changes.
- An unknown id, another project's video, or a **deleted** video is the same
  `404 not_found`.
- Renaming bumps `updatedAt`.

## Make a video public or protected (Coming soon)

> **Status: Coming soon.** The per-video **public** access mode is not released
> yet. Today every video is **protected**, and the `accessMode` field below does
> not work on a live account. This section describes how it will work. See
> [Feature status](/status).

Every video is **protected** today: it plays only with a signed, video-specific
token on your verified domains. When the mode ships, you will be able to switch
one video to **public** so it plays with no token on any site. See
[Protected and public videos](/playback-protection#protected-and-public-videos).

`PATCH /v1/projects/{projectId}/videos/{videoId}`

Body:

| Field | Required | Notes |
| --- | --- | --- |
| `accessMode` | No | Coming soon. `protected` (default) or `public`. |

```bash
curl -X PATCH "$BASE/v1/projects/$PROJECT_ID/videos/$VIDEO_ID" \
  -H "Authorization: Bearer $SEGMNT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"accessMode":"public"}'
```

Answer (`200`): the full `Video`, with the new `accessMode`.

- `protected` needs a signed, video-specific token. A token for one video never
  plays another. Public will play with no token — anyone with the link can share
  and embed it. You will be able to switch back to `protected` at any time.
- A missing body, an empty body, or a value that is not `protected` or `public`
  is `400 invalid_request`. Nothing changes.

## Delete a video

`DELETE /v1/projects/{projectId}/videos/{videoId}`

Delete when you no longer host a video, or to **free stored hours** on your plan
(stored hours fall shortly after the delete, not at once). Deleting a video is a
**soft delete** on our side, and it is permanent for you: the only way to bring
the video back is to **upload it again**, which creates a **new** `vid_...` id.
Update any links or records you kept.

```bash
curl -X DELETE "$BASE/v1/projects/$PROJECT_ID/videos/$VIDEO_ID" \
  -H "Authorization: Bearer $SEGMNT_KEY"
```

Answer: `204`, with no body.

What happens:

| Item | When |
| --- | --- |
| The original file in storage | Removed promptly. We keep it if the law requires, or for a legal matter. |
| Playback (also tokens issued before the delete) | Stops within about a minute. |
| Derived HLS renditions and thumbnails | Cleaned up shortly after. |
| The video list and `GET .../videos/{videoId}` | Hide the video right away. |
| A second `DELETE`, or a `GET` of the same video | `404 not_found`. |

Notes:

- Deleting a video **does not reset watched minutes**. It lowers your stored
  hours **shortly after** the delete, not instantly: we remove the original at
  once, and clean up the HLS renditions and the thumbnail in the background.
- Deleting a video **deletes the review frames with it**. We keep frames only
  while the video is under review, in an abuse report or in a legal matter;
  after the delete we keep no frame longer than **30 days**.
- Deleted videos are hidden from
  `GET /v1/projects/{projectId}/videos` and from the single-video `GET`
  immediately, and `PATCH` on them answers `404`.
- Retrying a `DELETE` is safe: the first call is `204`, the next is `404`.

## Code examples

> The rename, delete, retry-delete and error paths are covered by our test
> suite.

### Node.js (18+)

```js
const BASE = "https://api.segmnt.cloud";
const { SEGMNT_KEY, PROJECT_ID } = process.env;
const api = (path, init = {}) =>
  fetch(`${BASE}/v1/projects/${PROJECT_ID}${path}`, {
    ...init,
    headers: { Authorization: `Bearer ${SEGMNT_KEY}`, "Content-Type": "application/json", ...init.headers },
  });

export async function rename(videoId, title) {
  const res = await api(`/videos/${videoId}`, { method: "PATCH", body: JSON.stringify({ title }) });
  if (!res.ok) throw new Error(`rename failed: ${res.status} ${await res.text()}`);
  return res.json();
}

export async function remove(videoId) {
  const res = await api(`/videos/${videoId}`, { method: "DELETE" });
  if (res.status === 404) return false; // already deleted, or wrong id
  if (!res.ok) throw new Error(`delete failed: ${res.status} ${await res.text()}`);
  return true; // 204
}
```

### Python 3 (`pip install requests`)

```python
import os, requests

BASE = "https://api.segmnt.cloud"
API = f"{BASE}/v1/projects/{os.environ['PROJECT_ID']}"
HEADERS = {"Authorization": f"Bearer {os.environ['SEGMNT_KEY']}"}


def rename(video_id, title):
    r = requests.patch(f"{API}/videos/{video_id}", headers=HEADERS, json={"title": title})
    r.raise_for_status()
    return r.json()


def remove(video_id):
    r = requests.delete(f"{API}/videos/{video_id}", headers=HEADERS)
    if r.status_code == 404:
        return False  # already deleted, or wrong id
    r.raise_for_status()
    return True  # 204
```

### PHP / Laravel

```php
use Illuminate\Support\Facades\Http;

function segmntClient()
{
    return Http::withToken(config('services.segmnt.key'))->acceptJson();
}

function segmntRenameVideo(string $videoId, string $title): array
{
    $api = 'https://api.segmnt.cloud/v1/projects/' . config('services.segmnt.project');
    return segmntClient()->patch("$api/videos/$videoId", ['title' => $title])->throw()->json();
}

function segmntDeleteVideo(string $videoId): bool
{
    $api = 'https://api.segmnt.cloud/v1/projects/' . config('services.segmnt.project');
    $res = segmntClient()->delete("$api/videos/$videoId");
    if ($res->status() === 404) {
        return false; // already deleted, or wrong id
    }
    return $res->throw()->noContent(); // 204
}
```

## Errors

Errors use `application/problem+json`. Use the `code` field in your code, not
the text.

| Call | Status | `code` | Meaning | What to do |
| --- | --- | --- | --- | --- |
| rename | 400 | `invalid_request` | `title` is missing, empty, whitespace-only, longer than 200 characters, not a string, or the body is not JSON | Fix the body. Do not retry as-is. |
| delete | 401 | `unauthorized` | Missing, wrong or revoked key | Check the key. |
| any | 404 | `not_found` | Unknown video, another project's video, or an already deleted video | Check the id. A second `DELETE` lands here. |
| any | 405 | `method_not_allowed` | Wrong method for the URL | Use `PATCH` to rename, `DELETE` to delete. |

## Tested against the current release

Checked on 14 Oct 2026, after the
rename/delete route was released:

- `PATCH` and `DELETE .../videos/{videoId}` with no key answer
  `401 unauthorized`.
- `PUT` on the same URL answers `405 method_not_allowed`.
- The full path with a project key is proved by our API tests:
  `200` for a rename (and the trimmed title on
  `200` characters), `400 invalid_request` for a missing, empty, too-long or
  non-string title, `204` for a delete, `404` for the next `GET`, the second
  `DELETE` and an unknown or other-project video, and the deleted video is
  hidden from the list.
- The Node.js and Python examples above were run end to end against a local
  copy of the two routes (the same code; only the base URL changed).

We re-run these checks after every change to the route.

## Where to go next

- [Direct upload](/direct-upload) — the upload calls that create a video.
- [Usage and plan limits](/usage-and-limits) — how stored hours are counted.
- [Getting started](/getting-started) — the 5-minute path to your first video.
