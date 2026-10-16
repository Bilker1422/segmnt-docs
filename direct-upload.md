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
| URL life | Until `expiresAt` (24 hours after the start) |
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

Treat the part URLs like secrets. Do not log them.

In this version phone verification is **not required** to upload: a new free
account is **not** blocked with `403 phone_verification_required`. That code
returns only when the phone gate is switched on. A blocked account always gets
`403 account_banned`, and the ban check always applies. See
[Phone verification](/phone-verification).

## Step 2: PUT the parts

Cut the file into pieces of `partSizeBytes`. `PUT` the raw bytes of piece N to
the URL for `partNumber` N. No other header is needed.

Read the `ETag` response header and keep it with the part number.
Browsers can read `ETag` because our storage exposes it in CORS.

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
subscribe to the `video.ready` webhook. The webhook is **live**: register your
own HTTPS endpoint and we send the event when a video becomes ready. No customer
endpoint has received one yet, so test with your own receiver. See
[Feature status](/status).

In rare cases our team can hold a video for review (status `held`). It does not
play until the hold ends. See [Moderation and held videos](/moderation).

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
| create | 403 | `phone_verification_required` | Not returned in this version. When the phone gate is on: a new free account has not verified a phone number yet | Verify a phone number first ([Phone verification](/phone-verification)). |
| create | 403 | `account_banned` | The account is blocked | Contact us if you think this is a mistake. |
| any | 404 | `not_found` | Wrong project or upload ID, or the key is for another project | Check the IDs. |
| complete | 400 | `invalid_request` | `parts` list is not valid | Fix the request. |
| complete | 400 | `invalid_parts` | Part numbers or ETags do not match what was uploaded | Check that every part was uploaded and the ETags are right. |
| complete | 409 | `upload_closed` | Upload was cancelled or expired | Start a new upload. |
| cancel | 409 | `upload_closed` | Upload is already complete | Nothing to cancel. |
| create | 429 | (code not set in the spec yet) | Too many requests | Wait `Retry-After` seconds, then retry. |

Errors on the part `PUT` come from storage, not from our API. See the next section.

## Retry rules

| Call | Safe to retry? | How |
| --- | --- | --- |
| Create | Yes, with `Idempotency-Key` | Same key returns the same upload. Without a key, a retry makes a second video. |
| Part `PUT` | Yes | Send the same part again. The last one wins. Use the newest ETag. Retry on network errors, `5xx` and `429` with a growing wait (1 s, 2 s, 4 s). |
| Part `PUT` returns `403` | No | The URL expired. Start a new upload. |
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

In a browser, call "create" and "complete" from your **server** (the API key must
stay secret). Give the browser the `parts` list. The browser then `PUT`s each
`file.slice(start, end)` to its URL and reads `res.headers.get("ETag")`, and sends
the ETags back to your server.
