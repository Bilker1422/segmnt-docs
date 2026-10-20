---
title: Direct upload
description: Send a video straight to storage with pre-signed multipart URLs, then complete the upload. Node.js, Python and PHP/Laravel examples.
icon: "upload"
---

Your app sends the video file straight to our storage. Your server never carries
the file. The upload has 4 steps:

1. **Create** the upload. You get one pre-signed URL for each part.
2. **PUT** each part to its URL. Keep the `ETag` header of each answer.
3. **Complete** the upload with the list of part numbers and ETags.
4. We encode the video. Poll the video until it is `ready`, or subscribe to the `video.ready` webhook.

The pre-signed URLs are for upload only. Playback uses separate signed playback
tokens: see [Create playback tokens on your server](/playback-tokens) and
[Embed a video on any website](/embed-a-video).

## Before you start

- A project ID (`prj_...`) and an API key for that project (`sk_...`).
- Base URL: `https://api.segmnt.cloud`
- Send the key as `Authorization: Bearer <key>` on the API calls.
  Do **not** send it to the part URLs.

## Limits

| Item | Value |
| --- | --- |
| File types | `video/mp4`, `video/quicktime`, `video/webm`, `video/x-matroska` |
| Max file size | Set by your plan. The default is 5 GiB until plans are live. |
| Part size | `partSizeBytes` from the response. At least 100 MiB. |
| Max parts | 10,000 (the part size grows for very large files) |
| Last part | Can be smaller. All other parts must be exactly `partSizeBytes`. |
| Part URL life | 4 hours. The upload stays open until `expiresAt` (24 hours after the start). |
| Title | 1 to 200 characters |
| File name | 1 to 255 characters |

Small files (under 100 MiB) have one part.

The size limit is checked 3 times: when you create the upload (declared
`sizeBytes`), when you complete it (real size), and before encoding (real
duration). If the real file is over the limit, the video ends as `failed` with
`failure.code` = `plan_limit_reached`, and the file is removed.
If the real file is larger than the `sizeBytes` you declared, the video ends as
`failed` with `size_mismatch`.

## Step 1: Create the upload

```bash
curl -X POST "$BASE/v1/projects/$PROJECT/uploads" \
  -H "Authorization: Bearer $SEGMNT_KEY" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{"title":"Lesson 1","fileName":"lesson-01.mp4","sizeBytes":2147483648,"contentType":"video/mp4"}'
```

Answer (`201`):

```json
{
  "id": "upl_2bN6rD",
  "video": { "id": "vid_9xR4tL", "status": "uploading", "...": "..." },
  "partSizeBytes": 104857600,
  "parts": [
    { "partNumber": 1, "url": "https://..." },
    { "partNumber": 2, "url": "https://..." }
  ],
  "expiresAt": "2026-10-12T07:00:00Z"
}
```

`Idempotency-Key` is optional but we recommend it. If your request times out,
send it again with the same key (within 24 hours) and you get the same upload.
The answer has fresh part URLs, so you can send the parts again after the old
part URLs expire (part URLs work for 4 hours).

Treat the part URLs like secrets. Do not log them.

We do **not** ask for a phone number or an SMS code to upload. A blocked account
always gets `403 account_banned`, and the ban check always applies. See
[Phone verification](/phone-verification).

## Step 2: PUT the parts

Cut the file into pieces of `partSizeBytes`. `PUT` the raw bytes of piece N to
the URL for `partNumber` N. Send the part's exact size as `Content-Length`: the
part URL signs it, so a different length is refused. `Content-Type` is not
checked. No auth header is needed.

Read the `ETag` response header and keep it with the part number.
Browsers can read `ETag` because our storage exposes it in CORS. See
[Browser](#browser) for what a browser can and cannot do on your own domain.

You can send parts in parallel (4 at a time is a good start) and in any order.

## Step 3: Complete

```bash
curl -X POST "$BASE/v1/projects/$PROJECT/uploads/$UPLOAD/complete" \
  -H "Authorization: Bearer $SEGMNT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"parts":[{"partNumber":1,"etag":"\"abc...\""},{"partNumber":2,"etag":"\"def...\""}]}'
```

Send every part. Use the ETag exactly as you got it (keep the quote marks) —
quoted and unquoted ETags are both accepted. The ETag must come from the same
part number. If a part was sent twice, use the ETag of the newest answer.
Answer: `202` with the video, now with `status: "processing"`.

Complete is safe to retry. If the upload is already complete, you get `202` with
the current video.

## Step 4: Wait for the result

The video goes `processing` then `ready` (or `failed`). Poll
`GET /v1/projects/{projectId}/videos/{videoId}` until the status changes, or
subscribe to the [`video.ready` webhook](/webhooks). The webhook is **live**: register your
own HTTPS endpoint and we send the event when a video becomes ready. See
[Feature status](/status).

In rare cases our team can hold a video for review (status `held`). It does not
play until the hold ends. See [Moderation and held videos](/moderation).

Cancel only stops an upload that has **not** finished. To delete a video you
have already completed, see [Manage videos](/manage-videos#delete-a-video).

## Cancel an upload

```bash
curl -X DELETE "$BASE/v1/projects/$PROJECT/uploads/$UPLOAD" \
  -H "Authorization: Bearer $SEGMNT_KEY"
```

Answer: `204`. The parts and the video are removed. Calling it again is safe.
If the upload is already complete, you get `409 upload_closed`.
Open uploads that are not completed are removed after `expiresAt`.

## Errors

Errors use `application/problem+json`. Use the `code` field in your code, not the text.

| Where | Status | `code` | Meaning | What to do |
| --- | --- | --- | --- | --- |
| create | 400 | `invalid_request` | Missing or wrong field, or file type not supported | Fix the request. Do not retry. |
| any | 401 | `unauthorized` | Missing, wrong or revoked key | Check the key. |
| any | 405 | `method_not_allowed` | Wrong method for the URL | Check the URL and the HTTP method. |
| create | 403 | `plan_limit_reached` | File is larger than your plan allows, or you are over your stored-hours or new-upload limit | Use a smaller file, delete a video, or change plan. |
| create | 403 | `account_banned` | The account is blocked | Contact us if you think this is a mistake. |
| any | 404 | `not_found` | Wrong project or upload ID, or the key is for another project | Check the IDs. |
| complete | 400 | `invalid_request` | `parts` list is not valid | Fix the request. |
| complete | 400 | `invalid_parts` | Part numbers or ETags do not match what was uploaded | Check that every part was uploaded and the ETags are right. |
| complete | 409 | `upload_closed` | Upload was cancelled or expired | Start a new upload. |
| cancel | 409 | `upload_closed` | Upload is already complete | Nothing to cancel. |
| browser create/complete | 403 | `origin_mismatch` | The upload token is for another domain | Send the request from the verified domain the token names. |
| browser create/complete | 403 | `content_type_not_allowed` | The upload token does not allow this file type | Upload the content type the token allows. |
| browser create/complete | 409 | `token_used` | The upload token was already used, or it expired | Ask your server for a new token. |
| create | 429 | (code not set in the spec yet) | Too many requests | Wait `Retry-After` seconds, then retry. |

Errors on the part `PUT` come from storage, not from our API. See the next section.

## Retry rules

| Call | Safe to retry? | How |
| --- | --- | --- |
| Create | Yes, with `Idempotency-Key` | Same key returns the same upload. Without a key, a retry makes a second video. |
| Part `PUT` | Yes | Send the same part again. The last one wins. Use the newest ETag. Retry on network errors, `5xx` and `429` with a growing wait (1 s, 2 s, 4 s). |
| Part `PUT` returns `403` | No | The URL expired, or the part size did not match the signed `Content-Length`. Start a new upload. |
| Complete | Yes | Same body. If the answer was lost, send it again. |
| Cancel | Yes | Always `204` for a cancelled upload. |

Do not retry `400`, `401`, `403` and `404` from the API without a change.

## Code examples

> These examples were run against the current release. The Node, Python and PHP
> examples each
> uploaded a 250 MiB file (3 parts) and the video reached `processing`.
> The create, complete, cancel and error paths are also covered by our test
> suite.

### Node.js (18+)

```js
import { open } from "node:fs/promises";
import { randomUUID } from "node:crypto";

const BASE = "https://api.segmnt.cloud";
const { SEGMNT_KEY, PROJECT_ID } = process.env;
const api = (path, init = {}) =>
  fetch(`${BASE}/v1/projects/${PROJECT_ID}${path}`, {
    ...init,
    headers: { Authorization: `Bearer ${SEGMNT_KEY}`, "Content-Type": "application/json", ...init.headers },
  });

async function putPart(url, bytes) {
  for (let attempt = 0; attempt < 5; attempt++) {
    try {
      const res = await fetch(url, { method: "PUT", body: bytes });
      if (res.ok) return res.headers.get("etag");
      if (res.status !== 429 && res.status < 500) throw new Error(`part failed: ${res.status}`);
    } catch (e) {
      if (String(e.message).startsWith("part failed")) throw e;
    }
    await new Promise((r) => setTimeout(r, 1000 * 2 ** attempt));
  }
  throw new Error("part failed after retries");
}

export async function upload(path, title) {
  const file = await open(path);
  const { size } = await file.stat();
  const res = await api("/uploads", {
    method: "POST",
    headers: { "Idempotency-Key": randomUUID() },
    body: JSON.stringify({ title, fileName: path.split("/").pop(), sizeBytes: size, contentType: "video/mp4" }),
  });
  if (!res.ok) throw new Error(`create failed: ${res.status} ${await res.text()}`);
  const up = await res.json();
  try {
    const parts = [];
    for (const { partNumber, url } of up.parts) {
      const start = (partNumber - 1) * up.partSizeBytes;
      const length = Math.min(up.partSizeBytes, size - start);
      const buf = Buffer.alloc(length);
      await file.read(buf, 0, length, start);
      parts.push({ partNumber, etag: await putPart(url, buf) });
    }
    const done = await api(`/uploads/${up.id}/complete`, { method: "POST", body: JSON.stringify({ parts }) });
    if (!done.ok) throw new Error(`complete failed: ${done.status} ${await done.text()}`);
    return await done.json();
  } catch (e) {
    await api(`/uploads/${up.id}`, { method: "DELETE" });
    throw e;
  } finally {
    await file.close();
  }
}
```

### Python 3 (`pip install requests`)

```python
import os, time, uuid, requests

BASE = "https://api.segmnt.cloud"
PROJECT = os.environ["PROJECT_ID"]
HEADERS = {"Authorization": f"Bearer {os.environ['SEGMNT_KEY']}"}
API = f"{BASE}/v1/projects/{PROJECT}"


def put_part(url, data):
    for attempt in range(5):
        try:
            r = requests.put(url, data=data, timeout=300)
            if r.ok:
                return r.headers["ETag"]
            if r.status_code != 429 and r.status_code < 500:
                raise RuntimeError(f"part failed: {r.status_code}")
        except requests.RequestException:
            pass
        time.sleep(2 ** attempt)
    raise RuntimeError("part failed after retries")


def upload(path, title):
    size = os.path.getsize(path)
    r = requests.post(
        f"{API}/uploads",
        headers={**HEADERS, "Idempotency-Key": str(uuid.uuid4())},
        json={"title": title, "fileName": os.path.basename(path),
              "sizeBytes": size, "contentType": "video/mp4"},
    )
    r.raise_for_status()
    up = r.json()
    try:
        parts = []
        with open(path, "rb") as f:
            for p in up["parts"]:
                f.seek((p["partNumber"] - 1) * up["partSizeBytes"])
                data = f.read(up["partSizeBytes"])
                parts.append({"partNumber": p["partNumber"], "etag": put_part(p["url"], data)})
        done = requests.post(f"{API}/uploads/{up['id']}/complete", headers=HEADERS, json={"parts": parts})
        done.raise_for_status()
        return done.json()
    except Exception:
        requests.delete(f"{API}/uploads/{up['id']}", headers=HEADERS)
        raise
```

### PHP / Laravel

```php
use Illuminate\Support\Facades\Http;

function segmntUpload(string $path, string $title): array
{
    $api = 'https://api.segmnt.cloud/v1/projects/' . config('services.segmnt.project');
    $client = fn () => Http::withToken(config('services.segmnt.key'))->acceptJson();

    $size = filesize($path);
    $up = $client()
        ->withHeaders(['Idempotency-Key' => (string) \Illuminate\Support\Str::uuid()])
        ->post("$api/uploads", [
            'title' => $title,
            'fileName' => basename($path),
            'sizeBytes' => $size,
            'contentType' => 'video/mp4',
        ])->throw()->json();

    try {
        $parts = [];
        $file = fopen($path, 'rb');
        foreach ($up['parts'] as $p) {
            fseek($file, ($p['partNumber'] - 1) * $up['partSizeBytes']);
            $data = fread($file, $up['partSizeBytes']);
            // No Authorization header on the part URL.
            $res = Http::retry(5, 1000)->withBody($data, 'application/octet-stream')
                ->put($p['url'])->throw();
            $parts[] = ['partNumber' => $p['partNumber'], 'etag' => $res->header('ETag')];
        }
        fclose($file);

        return $client()->post("$api/uploads/{$up['id']}/complete", ['parts' => $parts])
            ->throw()->json();
    } catch (\Throwable $e) {
        $client()->delete("$api/uploads/{$up['id']}");
        throw $e;
    }
}
```

Note: this PHP example reads each part into memory (100 MiB or more). Use a
PHP `memory_limit` of at least 512 MB, or stream the part.

## Browser

A browser can upload **on your own domain** with a **browser upload token**. The
token is short-lived, single-use and capped in size. Your server creates it; the
browser never sees your API key.

The flow:

1. Your **server** calls `POST /v1/projects/{projectId}/upload-tokens` with your
   `full` API key. The body names the verified `domain`, a `maxSizeBytes` and the
   `contentType`.
2. Your **server** sends the returned `token` to the browser.
3. The **browser** calls `create` and `complete` with
   `Authorization: Bearer <token>` from that domain. It also `PUT`s each part to
   its pre-signed URL and reads the `ETag`.
4. The `PUT` work still goes straight to storage: the file never passes through
   the API.

The token allows exactly one `create` and its matching `complete`, and nothing
else: it cannot list, delete or play. A second `create` gets
`409 token_used`. The part `PUT` needs no auth header, and our storage allows
the `PUT` from any origin and exposes the `ETag` header.

Rules:

| Item | Value |
| --- | --- |
| Token life | 1 hour by default (`expiresInSeconds`, 300 to 7200 — 5 minutes to 2 hours) |
| `maxSizeBytes` | Defaults to your plan maximum. Capped at the plan maximum; larger gets `403 plan_limit_reached`. |
| `contentType` | The one file type the token allows: `video/mp4`, `video/quicktime`, `video/webm` or `video/x-matroska`. Omit it to allow any of the four. A different type gets `403 content_type_not_allowed`. |
| `domain` | A **verified** host of the project. A request from another `Origin` gets `403 origin_mismatch`. |
| Uses | One `create` and its `complete`. |

Create the token on your server:

```bash
curl -X POST "$BASE/v1/projects/$PROJECT_ID/upload-tokens" \
  -H "Authorization: Bearer $SEGMNT_KEY" \
  -H "Content-Type: application/json" \
  -d '{"domain":"courses.example.com","maxSizeBytes":524288000,"contentType":"video/mp4","title":"Lesson 1"}'
```

Answer (`201`):

```json
{
  "token": "utk_...",
  "expiresAt": "2026-10-19T13:00:00.000Z",
  "projectId": "prj_...",
  "domain": "courses.example.com",
  "maxSizeBytes": 524288000,
  "contentTypes": ["video/mp4"]
}
```

The browser then calls `create` and `complete` with the token instead of a key.
Bans, plan limits and the review hold (where it is on) still apply.

> If you cannot mint an upload token on your server, the older path still works:
> your **server** calls `create` and `complete`, and the browser only `PUT`s the
> parts. The browser cannot call `create` or `complete` with an **API key** from
> your own domain; only the token path above, or our dashboard origin, is
> allowed.
