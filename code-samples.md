---
title: "Code samples: upload, status, playback token and webhooks"
description: Copy-paste samples for the four calls most integrations need. Raw cURL first, then Node.js, Python and PHP. Each sample starts with a config block.
icon: "code"
---

Use these samples to call the API from your own code. Each one follows the same
order: the raw **cURL** call first, then **Node.js**, **Python** and **PHP**. Use
cURL to see exactly what goes over the wire. Use the other languages in your app.

Every sample starts with a **config block**. It names the values you set once:

| Variable | What it is | Where it comes from |
| --- | --- | --- |
| `SEGMNT_BASE_URL` | The API base URL. Set it to `https://api.segmnt.cloud`. | Fixed. |
| `SEGMNT_API_KEY` | Your project API key (`sk_...`). Keep it on your server. | Dashboard → project → **API keys**. |
| `PROJECT_ID` | Your project ID (`prj_...`). | Dashboard → project. |
| `SEGMNT_DOMAIN` | A domain you verified for the project, where the video plays (for example `courses.example.com`). Used only for playback tokens. | [Accounts, projects, API keys and domains](/accounts-and-projects). |
| `SEGMNT_WEBHOOK_SECRET` | The signing secret of one webhook endpoint (`whsec_...`). Used only to verify events. | Shown once when you add the endpoint. |

Every value in these samples is a placeholder. Replace `sk_YOUR_API_KEY`,
`prj_YOUR_PROJECT_ID` and `whsec_YOUR_WEBHOOK_SECRET` with your own values. Never
put a real key in code you share.

## 1. Upload a video (one part, up to 100 MiB)

**Status:** Available.

The samples below upload a file that is under 100 MiB. It has one part. For
larger files, see [Direct upload](/direct-upload), which splits the file into
parts.

The flow has three calls: **create** the upload, **PUT** the bytes to the part URL,
and **complete** the upload. The part URL is pre-signed. Do not send your API key to it.

If **complete** returns `429` with the code `encode_concurrency_limit`, another of
your videos is still processing. Wait until it is ready, then try again. The
samples stop with an error on any failed call.

### cURL

```bash
set -euo pipefail
export SEGMNT_BASE_URL="https://api.segmnt.cloud"
export SEGMNT_API_KEY="sk_YOUR_API_KEY"
export PROJECT_ID="prj_YOUR_PROJECT_ID"
export FILE="./lesson-01.mp4"

SIZE=$(wc -c < "$FILE" | tr -d ' ')

# 1. Create the upload. The answer holds the upload ID, the video ID and one part URL.
CREATE=$(curl -sS --fail-with-body -X POST "$SEGMNT_BASE_URL/v1/projects/$PROJECT_ID/uploads" \
  -H "Authorization: Bearer $SEGMNT_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"title\":\"Lesson 1\",\"fileName\":\"lesson-01.mp4\",\"sizeBytes\":$SIZE,\"contentType\":\"video/mp4\"}")
UPLOAD_ID=$(echo "$CREATE" | python3 -c 'import json,sys; print(json.load(sys.stdin)["id"])')
VIDEO_ID=$(echo "$CREATE" | python3 -c 'import json,sys; print(json.load(sys.stdin)["video"]["id"])')
PART_URL=$(echo "$CREATE" | python3 -c 'import json,sys; print(json.load(sys.stdin)["parts"][0]["url"])')

# 2. PUT the file to the part URL. Keep the ETag from the response header.
ETAG=$(curl -sS --fail-with-body -X PUT --data-binary "@$FILE" -D - -o /dev/null "$PART_URL" \
  | tr -d '\r' | awk -F': ' 'tolower($1)=="etag"{print $2}')

# 3. Complete the upload with the part number and the ETag.
curl -sS --fail-with-body -X POST "$SEGMNT_BASE_URL/v1/projects/$PROJECT_ID/uploads/$UPLOAD_ID/complete" \
  -H "Authorization: Bearer $SEGMNT_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"parts\":[{\"partNumber\":1,\"etag\":$(printf '%s' "$ETAG" | python3 -c 'import json,sys; print(json.dumps(sys.stdin.read()))')}]}"

echo "video: $VIDEO_ID"
```

### Node.js (18 or later)

```js
import { readFile } from "node:fs/promises";

const SEGMNT_BASE_URL = process.env.SEGMNT_BASE_URL ?? "https://api.segmnt.cloud";
const SEGMNT_API_KEY = process.env.SEGMNT_API_KEY;
const PROJECT_ID = process.env.PROJECT_ID;
const FILE = process.env.FILE ?? "./lesson-01.mp4";
if (!SEGMNT_API_KEY || !PROJECT_ID) throw new Error("Set SEGMNT_API_KEY and PROJECT_ID first.");

const api = (path, init = {}) =>
  fetch(`${SEGMNT_BASE_URL}/v1/projects/${PROJECT_ID}${path}`, {
    ...init,
    headers: { Authorization: `Bearer ${SEGMNT_API_KEY}`, "Content-Type": "application/json", ...init.headers },
  });

const bytes = await readFile(FILE);

// 1. Create the upload.
const created = await api("/uploads", {
  method: "POST",
  body: JSON.stringify({ title: "Lesson 1", fileName: "lesson-01.mp4", sizeBytes: bytes.length, contentType: "video/mp4" }),
});
if (created.status !== 201) throw new Error(`create failed: ${created.status} ${await created.text()}`);
const upload = await created.json();

// 2. PUT the bytes to the part URL. Keep the ETag.
const put = await fetch(upload.parts[0].url, { method: "PUT", body: bytes });
if (!put.ok) throw new Error(`part upload failed: ${put.status}`);
const etag = put.headers.get("etag");

// 3. Complete the upload.
const done = await api(`/uploads/${upload.id}/complete`, {
  method: "POST",
  body: JSON.stringify({ parts: [{ partNumber: 1, etag }] }),
});
if (done.status !== 202) throw new Error(`complete failed: ${done.status} ${await done.text()}`);

console.log("video:", upload.video.id);
```

### Python (3.9 or later, standard library only)

```python
import json
import os
import urllib.request
from pathlib import Path

SEGMNT_BASE_URL = os.environ.get("SEGMNT_BASE_URL", "https://api.segmnt.cloud")
SEGMNT_API_KEY = os.environ["SEGMNT_API_KEY"]
PROJECT_ID = os.environ["PROJECT_ID"]
FILE = Path(os.environ.get("FILE", "./lesson-01.mp4"))
# Python's default User-Agent is blocked by our firewall (error 1010). Send your own.
USER_AGENT = "segmnt-docs-sample/1.0"


def api(method, path, body=None):
    data = json.dumps(body).encode() if body is not None else None
    req = urllib.request.Request(
        f"{SEGMNT_BASE_URL}/v1/projects/{PROJECT_ID}{path}",
        data=data,
        method=method,
        headers={"Authorization": f"Bearer {SEGMNT_API_KEY}", "Content-Type": "application/json", "User-Agent": USER_AGENT},
    )
    with urllib.request.urlopen(req) as res:
        return res.status, json.loads(res.read() or b"null")


blob = FILE.read_bytes()

# 1. Create the upload.
status, upload = api("POST", "/uploads", {
    "title": "Lesson 1", "fileName": "lesson-01.mp4",
    "sizeBytes": len(blob), "contentType": "video/mp4",
})
assert status == 201, status

# 2. PUT the bytes to the part URL. Keep the ETag.
put = urllib.request.Request(upload["parts"][0]["url"], data=blob, method="PUT", headers={"User-Agent": USER_AGENT})
with urllib.request.urlopen(put) as res:
    etag = res.headers["ETag"]

# 3. Complete the upload.
status, _ = api("POST", f"/uploads/{upload['id']}/complete", {
    "parts": [{"partNumber": 1, "etag": etag}],
})
assert status == 202, status

print("video:", upload["video"]["id"])
```

### PHP (8.1 or later, with the cURL extension)

```php
<?php
$base = getenv('SEGMNT_BASE_URL') ?: 'https://api.segmnt.cloud';
$key = getenv('SEGMNT_API_KEY');
$project = getenv('PROJECT_ID');
$file = getenv('FILE') ?: './lesson-01.mp4';
if (!$key || !$project) {
    throw new RuntimeException('Set SEGMNT_API_KEY and PROJECT_ID first.');
}

function api(string $base, string $key, string $project, string $method, string $path, ?array $body = null): array
{
    $ch = curl_init("$base/v1/projects/$project$path");
    $headers = ["Authorization: Bearer $key", 'Content-Type: application/json'];
    curl_setopt_array($ch, [
        CURLOPT_CUSTOMREQUEST => $method,
        CURLOPT_HTTPHEADER => $headers,
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_HEADER => false,
    ]);
    if ($body !== null) {
        curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($body));
    }
    $raw = curl_exec($ch);
    $status = curl_getinfo($ch, CURLINFO_RESPONSE_CODE);
    curl_close($ch);
    return [$status, json_decode($raw, true)];
}

$bytes = file_get_contents($file);

// 1. Create the upload.
[$status, $upload] = api($base, $key, $project, 'POST', '/uploads', [
    'title' => 'Lesson 1', 'fileName' => 'lesson-01.mp4',
    'sizeBytes' => strlen($bytes), 'contentType' => 'video/mp4',
]);
if ($status !== 201) {
    throw new RuntimeException("create failed: $status");
}

// 2. PUT the bytes to the part URL. Read the ETag from the response headers.
$ch = curl_init($upload['parts'][0]['url']);
$etag = '';
curl_setopt_array($ch, [
    CURLOPT_CUSTOMREQUEST => 'PUT',
    CURLOPT_POSTFIELDS => $bytes,
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HEADERFUNCTION => function ($ch, $line) use (&$etag) {
        if (stripos($line, 'etag:') === 0) {
            $etag = trim(substr($line, 5));
        }
        return strlen($line);
    },
]);
curl_exec($ch);
curl_close($ch);

// 3. Complete the upload.
[$status] = api($base, $key, $project, 'POST', "/uploads/{$upload['id']}/complete", [
    'parts' => [['partNumber' => 1, 'etag' => $etag]],
]);
if ($status !== 202) {
    throw new RuntimeException("complete failed: $status");
}

echo "video: {$upload['video']['id']}\n";
```

## 2. Check the video status

A video starts as `uploading`, then `processing`, then `ready` or `failed`. Poll
the video until the status is `ready` or `failed`. Or subscribe to the
[`video.ready` webhook](/webhooks) and skip the polling.

Wait at least 5 seconds between checks. A video can take longer than a minute.

### cURL

```bash
set -euo pipefail
export SEGMNT_BASE_URL="https://api.segmnt.cloud"
export SEGMNT_API_KEY="sk_YOUR_API_KEY"
export PROJECT_ID="prj_YOUR_PROJECT_ID"
export VIDEO_ID="vid_YOUR_VIDEO_ID"

curl -sS --fail-with-body "$SEGMNT_BASE_URL/v1/projects/$PROJECT_ID/videos/$VIDEO_ID" \
  -H "Authorization: Bearer $SEGMNT_API_KEY"
```

### Node.js (18 or later)

```js
const SEGMNT_BASE_URL = process.env.SEGMNT_BASE_URL ?? "https://api.segmnt.cloud";
const { SEGMNT_API_KEY, PROJECT_ID, VIDEO_ID } = process.env;
if (!SEGMNT_API_KEY || !PROJECT_ID || !VIDEO_ID) throw new Error("Set SEGMNT_API_KEY, PROJECT_ID and VIDEO_ID first.");

async function videoStatus() {
  const res = await fetch(`${SEGMNT_BASE_URL}/v1/projects/${PROJECT_ID}/videos/${VIDEO_ID}`, {
    headers: { Authorization: `Bearer ${SEGMNT_API_KEY}` },
  });
  if (!res.ok) throw new Error(`status check failed: ${res.status}`);
  return (await res.json()).status;
}

let status;
do {
  await new Promise((r) => setTimeout(r, 5000));
  status = await videoStatus();
  console.log("status:", status);
} while (status !== "ready" && status !== "failed");
```

### Python (3.9 or later, standard library only)

```python
import json
import os
import time
import urllib.request

SEGMNT_BASE_URL = os.environ.get("SEGMNT_BASE_URL", "https://api.segmnt.cloud")
SEGMNT_API_KEY = os.environ["SEGMNT_API_KEY"]
PROJECT_ID = os.environ["PROJECT_ID"]
VIDEO_ID = os.environ["VIDEO_ID"]
USER_AGENT = "segmnt-docs-sample/1.0"  # Python's default User-Agent is blocked (error 1010).


def video_status():
    req = urllib.request.Request(
        f"{SEGMNT_BASE_URL}/v1/projects/{PROJECT_ID}/videos/{VIDEO_ID}",
        headers={"Authorization": f"Bearer {SEGMNT_API_KEY}", "User-Agent": USER_AGENT},
    )
    with urllib.request.urlopen(req) as res:
        return json.loads(res.read())["status"]


status = video_status()
print("status:", status)
while status not in ("ready", "failed"):
    time.sleep(5)
    status = video_status()
    print("status:", status)
```

### PHP (8.1 or later, with the cURL extension)

```php
<?php
$base = getenv('SEGMNT_BASE_URL') ?: 'https://api.segmnt.cloud';
$key = getenv('SEGMNT_API_KEY');
$project = getenv('PROJECT_ID');
$video = getenv('VIDEO_ID');
if (!$key || !$project || !$video) {
    throw new RuntimeException('Set SEGMNT_API_KEY, PROJECT_ID and VIDEO_ID first.');
}

do {
    sleep(5);
    $ch = curl_init("$base/v1/projects/$project/videos/$video");
    curl_setopt_array($ch, [
        CURLOPT_HTTPHEADER => ["Authorization: Bearer $key"],
        CURLOPT_RETURNTRANSFER => true,
    ]);
    $body = json_decode(curl_exec($ch), true);
    curl_close($ch);
    $status = $body['status'] ?? 'unknown';
    echo "status: $status\n";
} while (!in_array($status, ['ready', 'failed'], true));
```

## 3. Create a signed playback token

Create the token on your server, once for each time you show the video. Send the
domain where the video plays. The token works only on that domain. The answer
holds `token`, `expiresAt` and `embedUrl`.

The token is a playback credential. Send it to the browser only inside your page.
Do not put your API key in the browser.

### cURL

```bash
set -euo pipefail
export SEGMNT_BASE_URL="https://api.segmnt.cloud"
export SEGMNT_API_KEY="sk_YOUR_API_KEY"
export PROJECT_ID="prj_YOUR_PROJECT_ID"
export VIDEO_ID="vid_YOUR_VIDEO_ID"
export SEGMNT_DOMAIN="courses.example.com"

curl -sS --fail-with-body -X POST "$SEGMNT_BASE_URL/v1/projects/$PROJECT_ID/videos/$VIDEO_ID/playback-tokens" \
  -H "Authorization: Bearer $SEGMNT_API_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"domain\":\"$SEGMNT_DOMAIN\"}"
```

### Node.js (18 or later)

```js
const SEGMNT_BASE_URL = process.env.SEGMNT_BASE_URL ?? "https://api.segmnt.cloud";
const { SEGMNT_API_KEY, PROJECT_ID, VIDEO_ID, SEGMNT_DOMAIN } = process.env;
if (!SEGMNT_API_KEY || !PROJECT_ID || !VIDEO_ID || !SEGMNT_DOMAIN) throw new Error("Set SEGMNT_API_KEY, PROJECT_ID, VIDEO_ID and SEGMNT_DOMAIN first.");

const res = await fetch(`${SEGMNT_BASE_URL}/v1/projects/${PROJECT_ID}/videos/${VIDEO_ID}/playback-tokens`, {
  method: "POST",
  headers: { Authorization: `Bearer ${SEGMNT_API_KEY}`, "Content-Type": "application/json" },
  body: JSON.stringify({ domain: SEGMNT_DOMAIN }),
});
if (res.status !== 201) throw new Error(`token failed: ${res.status} ${await res.text()}`);
const { token, expiresAt } = await res.json();
console.log("expires:", expiresAt, "token length:", token.length);
```

### Python (3.9 or later, standard library only)

```python
import json
import os
import urllib.request

SEGMNT_BASE_URL = os.environ.get("SEGMNT_BASE_URL", "https://api.segmnt.cloud")
SEGMNT_API_KEY = os.environ["SEGMNT_API_KEY"]
PROJECT_ID = os.environ["PROJECT_ID"]
VIDEO_ID = os.environ["VIDEO_ID"]
SEGMNT_DOMAIN = os.environ["SEGMNT_DOMAIN"]
USER_AGENT = "segmnt-docs-sample/1.0"  # Python's default User-Agent is blocked (error 1010).

req = urllib.request.Request(
    f"{SEGMNT_BASE_URL}/v1/projects/{PROJECT_ID}/videos/{VIDEO_ID}/playback-tokens",
    data=json.dumps({"domain": SEGMNT_DOMAIN}).encode(),
    method="POST",
    headers={"Authorization": f"Bearer {SEGMNT_API_KEY}", "Content-Type": "application/json", "User-Agent": USER_AGENT},
)
with urllib.request.urlopen(req) as res:
    body = json.loads(res.read())

print("expires:", body["expiresAt"], "token length:", len(body["token"]))
```

### PHP (8.1 or later, with the cURL extension)

```php
<?php
$base = getenv('SEGMNT_BASE_URL') ?: 'https://api.segmnt.cloud';
$key = getenv('SEGMNT_API_KEY');
$project = getenv('PROJECT_ID');
$video = getenv('VIDEO_ID');
$domain = getenv('SEGMNT_DOMAIN');
if (!$key || !$project || !$video || !$domain) {
    throw new RuntimeException('Set SEGMNT_API_KEY, PROJECT_ID, VIDEO_ID and SEGMNT_DOMAIN first.');
}

$ch = curl_init("$base/v1/projects/$project/videos/$video/playback-tokens");
curl_setopt_array($ch, [
    CURLOPT_POST => true,
    CURLOPT_HTTPHEADER => ["Authorization: Bearer $key", 'Content-Type: application/json'],
    CURLOPT_POSTFIELDS => json_encode(['domain' => $domain]),
    CURLOPT_RETURNTRANSFER => true,
]);
$raw = curl_exec($ch);
$status = curl_getinfo($ch, CURLINFO_RESPONSE_CODE);
curl_close($ch);

if ($status !== 201) {
    throw new RuntimeException("token failed: $status");
}
$body = json_decode($raw, true);
echo "expires: {$body['expiresAt']}\n";
```

## 4. Verify a webhook signature

Each delivery carries a `Segmnt-Signature` header:

```text
Segmnt-Signature: t=<unix seconds>,v1=<64 hex characters>
```

The `v1` value is the HMAC-SHA256 of `<t>.<raw body>`. The key is your signing secret,
used exactly as you got it, with the `whsec_` prefix. Check the signature against
the **raw request body**, before you parse the JSON. Reject a timestamp more than
300 seconds from the current time.

The cURL sample below builds a signed sample delivery. Use it to check your own
receiver. It does not call our servers.

### cURL

```bash
set -euo pipefail
export SEGMNT_WEBHOOK_SECRET="whsec_YOUR_WEBHOOK_SECRET"
export RECEIVER_URL="https://your-server.example/segmnt/webhook"

BODY='{"id":"evt_test_1","type":"video.ready"}'
T=$(date +%s)
V1=$(printf '%s.%s' "$T" "$BODY" | openssl dgst -sha256 -hmac "$SEGMNT_WEBHOOK_SECRET" | awk '{print $NF}')

curl -sS --fail-with-body -X POST "$RECEIVER_URL" \
  -H "Content-Type: application/json" \
  -H "Segmnt-Signature: t=$T,v1=$V1" \
  --data-binary "$BODY"
```

### Node.js (18 or later)

```js
import { createHmac, timingSafeEqual } from "node:crypto";

const SEGMNT_WEBHOOK_SECRET = process.env.SEGMNT_WEBHOOK_SECRET;
if (!SEGMNT_WEBHOOK_SECRET) throw new Error("Set SEGMNT_WEBHOOK_SECRET first.");

// rawBody is the request body as bytes (a Buffer), not the parsed JSON.
export function verifySegmntSignature(secret, header, rawBody, toleranceSeconds = 300) {
  const m = /^t=(\d+),v1=([0-9a-f]{64})$/.exec(header ?? "");
  if (!m) return false;
  const t = Number(m[1]);
  if (Math.abs(Date.now() / 1000 - t) > toleranceSeconds) return false;
  const expected = createHmac("sha256", secret).update(`${t}.`).update(rawBody).digest();
  const given = Buffer.from(m[2], "hex");
  return given.length === expected.length && timingSafeEqual(given, expected);
}
```

Call it in your route with the raw body. In Express, use `express.raw({ type: "application/json" })`
on this route, and read `req.headers["segmnt-signature"]`.

### Python (3.9 or later, standard library only)

```python
import hashlib
import hmac
import os
import re
import time

SEGMNT_WEBHOOK_SECRET = os.environ["SEGMNT_WEBHOOK_SECRET"]


def verify_segmnt_signature(secret: str, header: str | None, raw_body: bytes, tolerance: int = 300) -> bool:
    m = re.fullmatch(r"t=(\d+),v1=([0-9a-f]{64})", header or "")
    if not m:
        return False
    t = int(m.group(1))
    if abs(time.time() - t) > tolerance:
        return False
    expected = hmac.new(secret.encode(), f"{t}.".encode() + raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, m.group(2))
```

Pass `request.get_data()` (Flask) or `request.body` (Django) as `raw_body`. Do not pass the parsed JSON.

### PHP (8.1 or later)

```php
<?php
$secret = getenv('SEGMNT_WEBHOOK_SECRET');
if (!$secret) {
    throw new RuntimeException('Set SEGMNT_WEBHOOK_SECRET first.');
}

function verify_segmnt_signature(string $secret, ?string $header, string $rawBody, int $tolerance = 300): bool
{
    if (!preg_match('/^t=(\d+),v1=([0-9a-f]{64})$/', $header ?? '', $m)) {
        return false;
    }
    $t = (int) $m[1];
    if (abs(time() - $t) > $tolerance) {
        return false;
    }
    $expected = hash_hmac('sha256', $t . '.' . $rawBody, $secret);
    return hash_equals($expected, $m[2]);
}

// Read the raw body before anything parses it.
$rawBody = file_get_contents('php://input');
$ok = verify_segmnt_signature($secret, $_SERVER['HTTP_SEGMNT_SIGNATURE'] ?? null, $rawBody);
http_response_code($ok ? 200 : 401);
```

## Next steps

- [Direct upload](/direct-upload): files over 100 MiB, with several parts.
- [Create playback tokens on your server](/playback-tokens): when and how often to mint a token.
- [Webhooks](/webhooks): the event list, retries and the delivery log.
- [Errors](/errors): what each status code means.
